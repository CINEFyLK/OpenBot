# CINEFy MD — WhatsApp Automation Bot

A multi-provider WhatsApp automation bot built on the [`amiudmodz`](https://www.npmjs.com/package/amiudmodz) fork of [Baileys](https://github.com/WhiskeySockets/Baileys), with an Express control plane for pairing, session lifecycle management, and live status.

> **Runtime:** Node.js 18+ · ESM (`"type": "module"`) · top-level `await` entry point
> **Package name:** `angel-glitchers-bot` · **License:** ISC

---

## Table of Contents

- [Architecture](#architecture)
- [Requirements](#requirements)
- [Installation](#installation)
- [Configuration](#configuration)
- [Boot Sequence](#boot-sequence)
- [Connection Lifecycle](#connection-lifecycle)
- [Session Persistence](#session-persistence)
- [Control Plane API](#control-plane-api)
- [Command System](#command-system)
- [Command Reference](#command-reference)
- [Downloader Engine](#downloader-engine)
- [Movie Engine](#movie-engine)
- [AI Layer](#ai-layer)
- [Logging](#logging)
- [Project Layout](#project-layout)
- [Extending](#extending)
- [Deployment](#deployment)
- [Security Notes](#security-notes)

---

## Architecture

The system is split into two long-lived subsystems that share a single process:

```
┌──────────────────────── Node.js process ────────────────────────┐
│                                                                 │
│  ┌──────────────────────────┐        ┌──────────────────────┐  │
│  │  Control Plane (Express) │        │  Data Plane (Baileys)│  │
│  │  src/api/server.js       │        │  src/bot.js          │  │
│  │  src/api/routes.js       │◄──────►│  src/events/*        │  │
│  │  src/dashboard/**        │ import │  src/commands/**     │  │
│  └──────────────────────────┘  calls └──────────┬───────────┘  │
└─────────────────────────────────────────────────┼──────────────┘
                                                  │
                              ┌───────────────────┼───────────────────┐
                              ▼                   ▼                   ▼
                        aiudmodz socket      downloader engine      movie engine
                       (WhatsApp Web)         (yt-dlp + ffmpeg)     (HTTP API)
```

**Control plane** — an Express server exposing bot lifecycle verbs over `/api/bot`, plus a static dashboard. It imports the bot module directly and mutates socket state in-process; there is no queue, broker, or IPC hop.

**Data plane** — a single `makeWASocket()` instance wired to a recursive command loader. Message handling is entirely pull-through: the socket emits `messages.upsert`, the dispatcher extracts text, resolves a command from a `Map`, and invokes it.

Key architectural properties:

- **Single socket, single owner.** One module-scoped `sock` variable; all lifecycle functions mutate or replace it. There is no multi-session support.
- **Lazy command registry.** Commands are dynamically `import()`ed at startup and cached in a `Map` keyed by lowercase name plus every alias. A `commandsReady` promise is awaited at the top of every message batch so no message races an incomplete registry.
- **In-memory state.** The movie engine, downloader search state, and the Baileys version cache all live in module scope and are lost on restart. Nothing but the auth directory is persisted.
- **Best-effort reconnection.** Disconnects are classified by `DisconnectReason`; transient codes self-heal on a timer, terminal codes (`loggedOut`) stop the loop and require re-pairing.

---

## Requirements

| Component | Version | Notes |
|-----------|---------|-------|
| Node.js | ≥ 18 | ESM + top-level await |
| `yt-dlp` | latest | Required by all downloader commands except `mediafire` |
| `ffmpeg` | any | Optional; enables MP3 transcoding (audio falls back to M4A/WebM/Opus) |
| Python | 3.x | Only if invoking yt-dlp as `py -m yt_dlp` |

Install the external binaries:

```bash
# yt-dlp (pick one)
py -m pip install -U yt-dlp      # Windows
python3 -m pip install -U yt-dlp # Linux/macOS
# or: winget install yt-dlp

# ffmpeg
winget install Gyan.FFmpeg       # Windows
sudo apt install ffmpeg          # Debian/Ubuntu
brew install ffmpeg              # macOS
```

A bundled `@ffmpeg-installer/ffmpeg` binary is present as an npm dependency, but `src/engines/downloader/downloader.engine.js` probes the **system** `ffmpeg` on `PATH` — the npm binary is not wired into the probe.

---

## Installation

```bash
git clone <your-repo-url>
cd angel-glitchers-bot
npm install
cp .env.example .env    # then edit
npm start
```

The dashboard is served at `http://localhost:${PORT:-3000}`.

---

## Configuration

All configuration is environment-driven and parsed once in `config.js`.

| Variable | Default | Purpose |
|----------|---------|---------|
| `PORT` | `3000` | Express listen port (`parseInt(...) \|\| 3000`) |
| `OWNER_NUMBER` | — | Owner number, normalized to digits only |
| `BOT_PREFIX` | `.` | Command prefix; matched with `text.startsWith()` |
| `AUTH_DIR` | `./auth` | Resolved relative to project root via `path.join(rootDir, ...)` |
| `GEMINI_API_KEY` | — | `.ai`, `.kyrexi` |
| `MISTRAL_API_KEY` | — | `.mistral` |
| `GROQ_API_KEY` | — | `.groq` |
| `OPENROUTER_API_KEY` | — | `.openrouter` |
| `CEREBRAS_API_KEY` | — | `.cerebra`, `.imaginecerebra` |
| `KYREXI_API_KEY` | — | `.kyrexi` |
| `POLLINATIONS_API_KEY` / `POLLINATIONS_KEY` | — | All Pollinations-routed chat and image commands; first non-empty wins |

```bash
PORT=3000
OWNER_NUMBER=94706000390
BOT_PREFIX=.
AUTH_DIR=./auth
GEMINI_API_KEY=...
MISTRAL_API_KEY=...
GROQ_API_KEY=...
OPENROUTER_API_KEY=...
CEREBRAS_API_KEY=...
KYREXI_API_KEY=...
POLLINATIONS_API_KEY=...
```

> AI keys are **optional**. Each command reads its own key at call time, so you can enable only the providers you have keys for. Missing keys surface as command-level failures caught by the dispatcher, which replies `❌ Command failed.`

**Normalization:** `OWNER_NUMBER` is stripped of all non-digits at load (`replace(/\D/g, '')`), so `+94 70 600 0390` and `94706000390` are equivalent.

---

## Boot Sequence

`index.js` performs three steps at module top level, in order:

1. **Create and bind the control plane.** `createServer()` returns a configured Express app; `.listen(config.port)` logs the dashboard URL.
2. **Probe for an existing session.** `isPaired()` reads `<AUTH_DIR>/creds.json` and returns `JSON.parse(raw)?.registered === true`. A missing or malformed file is swallowed and treated as unpaired.
3. **Start the data plane.** `startBot()` creates the auth state directory, loads multi-file credentials, resolves the Baileys version, and constructs the socket.

Because step 3 is awaited at top level, a fatal error during socket construction rejects the module and terminates the process.

### Socket construction

```js
makeWASocket({
  version:                 await getBaileysVersion(),  // cached 6h
  auth:                    state,                       // useMultiFileAuthState
  logger,                                        // Baileys-compatible logger
  browser:                 Browsers.ubuntu('Chrome'),
  connectTimeoutMs:        60000,
  keepAliveIntervalMs:     15000,
  markOnlineOnConnect:     true,
  generateHighQualityLinkPreview: true,
  shouldSyncHistoryMessage: () => !isRegistered,
  syncFullHistory:         false,
  antiban:                 false
})
```

Two flags deserve attention:

- `shouldSyncHistoryMessage: () => !isRegistered` is a **predicate evaluated per sync decision**, not a boolean. History is only pulled until the first successful registration, then the closure variable flips to `true` and history sync stops permanently for that socket.
- `antiban: false` disables Baileys' evasion heuristics. This reduces latency and log noise but increases the risk of a `401`/`403` disconnect on aggressive usage patterns.

### Version cache

`fetchLatestBaileysVersion()` is a network call, so it is memoized:

- `VERSION_TTL_MS = 6 * 60 * 60 * 1000` (6 hours)
- On `DisconnectReason.restartRequired` or `connectionLost`, the cache is invalidated (`version = null`, `versionFetchedAt = 0`) so the next reconnect re-resolves against current WhatsApp Web. This is the correct response to those codes — a stale version is the usual root cause.

---

## Connection Lifecycle

State is tracked in a module-scoped object and projected through `getStatus()`:

```js
status = { state: 'offline', registered: false, connectedJid: null, lastError: null }
```

`state` transitions: `offline → connecting → open → offline`.

### Event wiring

| Event | Handler behavior |
|-------|------------------|
| `creds.update` | `saveCreds` — persists auth files |
| `messages.upsert` | Ignored unless `status.state === 'open'`; delegates to `handleMessage` inside try/catch |
| `connection.update` (`qr`) | Logs pairing hint pointing at the dashboard |
| `connection.update` (`open`) | Captures `isRegistered`, records JID, clears `lastError` |
| `connection.update` (`close`) | Classifies error, updates status, schedules reconnect or halts |

### Disconnect classification

The close handler unwraps the HTTP status code from either a `Boom` instance (`error.output.statusCode`) or a plain error (`error.output?.statusCode || error?.code`):

| Code | Action |
|------|--------|
| `DisconnectReason.loggedOut` | **Terminal.** No reconnect. Session must be re-paired via dashboard. |
| `DisconnectReason.restartRequired` | Invalidate version cache. Reconnect after **1s**. |
| `DisconnectReason.connectionLost` | Invalidate version cache. Reconnect after **5s**. |
| Any other code | Reconnect after **5s**. |
| `stopping === true` | No reconnect regardless of code — set by `disconnectBot()`. |

The reconnect timer is single-slot: every call does `clearTimeout(reconnectTimer)` before assigning, so overlapping disconnect events cannot stack reconnect loops. `startBot()` is recursive via `setTimeout`, and its rejection is caught and logged rather than crashing the process.

### `waitForConnectionOpen`

Pairing requires an open WebSocket, so `requestPairingCode` awaits a promise that resolves on `connection === 'open'` and rejects on `'close'`:

- Default timeout **30s**, after which it rejects with `Timed out waiting for WhatsApp to connect`.
- If `sock?.ws?.isOpen` is already true, it resolves synchronously.
- The handler removes itself from `sock.ev` in a `cleanup()` closure called on **both** settle paths and on timeout — no listener leak across repeated pairing attempts.

---

## Session Persistence

Credentials use Baileys' `useMultiFileAuthState`, which writes a set of files under `AUTH_DIR` and rewrites them on every `creds.update`.

Three distinct teardown semantics:

| Operation | Socket | `AUTH_DIR` | Auto-reconnect |
|-----------|--------|------------|----------------|
| `disconnectBot()` | `sock.end(undefined)` | **preserved** | suppressed via `stopping` |
| `logoutBot()` | closed | **`fs.rm(recursive, force)`** | suppressed |
| `restartBot()` | closed | preserved | resumes via `startBot()` |

`AUTH_DIR` is git-ignored. Anyone with read access to it holds a fully authenticated WhatsApp session — treat it as a credential, not a cache.

---

## Control Plane API

Express app assembled in `src/api/server.js`:

- `express.json()` body parsing
- `Router` mounted at `/api/bot`
- `express.static` serving `src/dashboard`
- Catch-all `app.get('*')`: paths under `/api` return `404 { error: 'Endpoint not found' }`; everything else serves `src/dashboard/pages/index.html` (SPA fallback)

| Method | Endpoint | Body | Response |
|--------|----------|------|----------|
| `GET` | `/api/bot/status` | — | `getStatus()` snapshot |
| `POST` | `/api/bot/pair` | `{ phoneNumber }` | `{ success, pairingCode }` |
| `POST` | `/api/bot/restart` | — | `{ success, status }` |
| `POST` | `/api/bot/disconnect` | — | `{ success, status }` |
| `POST` | `/api/bot/logout` | — | `{ success, status }` |

All mutating routes are wrapped in try/catch and return `500` with `{ error: err.message }`.

`GET /api/bot/status` payload:

```json
{
  "state": "open",
  "online": true,
  "registered": true,
  "paired": true,
  "jid": "94706000390@s.whatsapp.net",
  "error": null
}
```

`paired` reads the live socket (`sock?.authState?.creds?.registered === true`) while `registered` reflects the last known disk-backed value — they diverge during a disconnect.

### Pairing flow

`POST /api/bot/pair` strips non-digits and rejects inputs shorter than 8 characters (`Enter a full phone number with country code.`). If the socket is not open it calls `startBot()` first, then `waitForConnectionOpen()` if unregistered, then `sock.requestPairingCode(phone)`.

The resulting 8-digit code is entered on the phone under **Settings → Linked devices → Link a device**.

> The dashboard has **no authentication**. Every route — including `/logout`, which destroys the session — is reachable by anything that can reach the port. Bind it to loopback or put it behind an authenticating reverse proxy. See [Security Notes](#security-notes).

---

## Command System

### Loading

`src/events/messages.js` walks `src/commands` recursively:

- Directories are traversed depth-first; only `.js` files are considered.
- Each file is loaded with a dynamic `import()` of an absolute `file://` URL (backslashes normalized to forward slashes for Windows).
- A file whose default export lacks `name` or `run` is skipped — this is how shared modules like `src/commands/ai/_shared.js` and the `createDownloadCommand` factory in `src/commands/downloader/_downloadCommand.js` coexist with real commands.
- **A failed import is logged and skipped, not fatal.** One broken command file will not prevent startup.
- Every alias is registered as an independent key pointing at the same command object, all lowercased.
- Completion resolves `commandsReady`, which logs `Loaded N commands.`

Because every `.js` file under `src/commands` is imported, a syntax error in any of them produces a load-time log line rather than a crash — but the affected command is silently absent.

### Dispatch

For each message in a `messages.upsert` batch:

1. **Skip self-sent and empty messages.** `message.key.fromMe` and falsy `message.message` both bail.
2. **Extract text** via `normalizeMessageContent(rawMessage)`, falling back to the raw message if normalization returns nothing. Checked in order: `conversation` → `extendedTextMessage.text` → `imageMessage.caption` → `videoMessage.caption` → `documentMessage.caption`. **Captions on media are therefore valid command input.**
3. **Strip the prefix** if `text.startsWith(config.prefix)`; an empty remainder bails.
4. **Split on whitespace**, discarding empties. `args` excludes the command name.
5. **Resolve** via `resolveCommand`, which applies the numeric-reply rules below.
6. **Check `middleware`** if present; falsy return short-circuits with `❌ Owner only command.`
7. **Invoke `command.run(sock, sender, args, message)`** inside try/catch, replying `❌ Command failed.` on throw.

`sender` is `message.key.remoteJid` (the chat), while `participant` is `message.key.participant || sender` (the individual author in groups). **Middleware receives `participant`, but `run` receives `sender`** — owner checks are per-user, responses go to the chat.

### Interactive numeric replies

`resolveCommand` adds prefix-free routing so users can answer prompts with a bare number:

```js
if (!command && !isCmd && /^[1-9]\d*$/.test(text) && movieEngine.hasActiveSession(participant))
    → commands.get('movie'), args = [text, ...args]

if (!command && !isCmd && /^[1-5]$/.test(text))
    → commands.get('ytmp4') || commands.get('ytmp3') || commands.get('song')
```

Consequences worth knowing:

- Movie navigation accepts **any** positive integer, gated on a live session. Downloader selection accepts only `1`–`5`, matching the five-result search cap, and requires no session — so **any bare `1`–`5` in any chat triggers a YouTube download attempt** regardless of whether a prompt preceded it.
- Bare numerals can never reach a same-named command; the numeric branch takes precedence.

### Command contract

```js
export default {
  name: 'example',                  // required, registry key
  aliases: ['ex', 'e'],             // optional, each becomes a key
  desc: 'Human readable description',
  category: 'general',              // metadata only — not enforced anywhere
  middleware: (participant, owner) => boolean,   // optional gate
  run: async (sock, sender, args, msg) => { ... }
};
```

`category` is **purely descriptive**. It is never read by the dispatcher, and the bundled `menu` command renders a hardcoded string rather than introspecting the registry. There is currently no dynamic menu.

The `middleware` hook is supported by the dispatcher but **no bundled command uses it** — see [Security Notes](#security-notes).

---

## Command Reference

### AI

| Command | Aliases | Model / route |
|---------|---------|---------------|
| `.chat` | `pollinations` | `openai` via Pollinations |
| `.chatgpt` | `gpt`, `openai` | `openai` via Pollinations |
| `.chatgemini` | `gemini`, `googleai` | `gemini-3-flash` via Pollinations |
| `.chatclaude` | `claude`, `anthropic` | `claude-large` via Pollinations |
| `.chatllama` | `llama`, `metaai` | `llama` via Pollinations |
| `.chatqwen` | `qwen`, `qwenai` | `qwen-coder` via Pollinations |
| `.groq` | `gq`, `bro` | `llama-3.3-70b-versatile` (Groq API) |
| `.mistral` | `mKariya`, `mk` | `mistral-small-latest` (`@mistralai/mistralai`) |
| `.openrouter` | — | `openrouter/free` |
| `.cerebra` | `cere`, `sweet` | `llama3.1-70b` |
| `.kyrexi` | `kyra`, `sweetie` | `gemini-2.5-flash` (`@google/genai`) |
| `.ai` | `kariya`, `ask` | `gemini-2.5-flash` (`@google/genai`) |
| `.meta` | `askmeta`, `metatag` | utility |
| `.imagine` | `draw`, `paint`, `img` | Pollinations image endpoint |
| `.imaginecerebra` | `icere`, `cdraw`, `cai` | Cerebras image |

### Downloader

| Command | Type | Notes |
|---------|------|-------|
| `.ytmp4` | video | Search or direct link |
| `.ytmp3` | audio | MP3 when ffmpeg present |
| `.ytm3` | audio | Variant |
| `.song` | audio | Variant |
| `.tiktok` | video | Link required |
| `.fb` | video | Link required (facebook / fb.watch) |
| `.mediafire` | file | Direct link scrape, no yt-dlp |

### General & Utility

`menu` · `ping` · `alive` · `jid` · `getdp` (`profilepic`, `pp`, `getpic`) · `vo` (`viewonce`, `retrieve`) · `upload` (`url`, `tourl`) · `compress` · `document` · `imagehost` (`hostimage`) · `longlink` (`makelong`, `obscureurl`, `biglink`) · `reveal` · `movie` (`mv`, `downloadmovie`, `moviebox`, `movieboxdl`) · `cinesubz` (`cinetv`) · `send` (`sendtext`) · `send_payload` (`payload_cinefy`, `payload_send`)

### Group

| Command | File | Notes |
|---------|------|-------|
| `.kick` | `groups/kick.js` | Resolves target from reply `contextInfo.participant`, else first `mentionedJid` |
| `.hidetag` | `groups/hidetag.js` | Mentions all participants |
| `.tagall` | `groups/welcome.js` | Note: the command name is `tagall`, not `welcome` |

`.kick` calls `sock.groupParticipantsUpdate(sender, [target], 'remove')` and therefore only works when invoked in a group, with the bot as admin.

---

## Downloader Engine

`src/engines/downloader/downloader.engine.js` shells out to `yt-dlp` via `child_process.spawn`.

### Binary resolution

Three candidates are tried in order:

```js
[ { command: 'py',       args: ['-m', 'yt_dlp'] },
  { command: 'python',   args: ['-m', 'yt_dlp'] },
  { command: 'yt-dlp',   args: [] } ]
```

The loop advances to the next candidate **only** when the error message matches `/not recognized|ENOENT|No module named yt_dlp/i` — i.e. the binary is absent, not when it ran and failed. A genuine download error propagates immediately. If all three are exhausted, the thrown message names the last error, and the command layer surfaces an install hint.

### Platform detection

```js
/tiktok\.com|vm\.tiktok\.com/i                  → 'tiktok'
/facebook\.com|fb\.watch|fb\.com/i              → 'facebook'
/youtu\.be|youtube\.com|music\.youtube\.com/i    → 'youtube'
otherwise                                       → 'youtube'
```

Non-URL input on the YouTube platform is rewritten to a yt-dlp search prefix — `ytsearch5:` for `searchMedia`, `ytsearch1:` for `downloadMedia`.

### Metadata probe

```bash
yt-dlp --no-check-certificates --dump-single-json --no-playlist --no-warnings <target>
```

`normalizeInfo` unwraps a playlist by taking the first truthy entry when `entries` is an array. `bestThumbnail` sorts `info.thumbnails` by `width * height` descending and falls back to `info.thumbnail`.

> `--no-check-certificates` disables TLS validation on every outbound fetch. It is present because some hosts present broken chains, but it removes the guarantee that the download came from the host you asked for.

### Format selection and size caps

| Mode | Caps | Selector |
|------|------|----------|
| Audio (ffmpeg) | 25 MB | `bestaudio/best` + `-x --audio-format mp3 --audio-quality 0` |
| Audio (fallback) | 25 MB | `bestaudio[ext=m4a]/bestaudio/best` (no transcode) |
| Video | 64 MB | `b[ext=mp4][height<=480]/bv*[ext=mp4][height<=480]+ba[ext=m4a]/b[ext=mp4]/best` + `--merge-output-format mp4` |

`MAX_AUDIO_SIZE = 25 * 1024 * 1024` and `MAX_VIDEO_SIZE = 64 * 1024 * 1024` are enforced twice: as `--max-filesize`, and again with an explicit `fs.stat` check that reports the real size in MB.

The ffmpeg probe (`ffmpeg -version`) is memoized in `ffmpegAvailable`. If the MP3 transcode path throws, the job directory is wiped and recreated before falling back to the container-native download — so a partial transcode cannot be picked up by `findDownloadedFile`.

### Job lifecycle

1. `jobsDir = .cache/downloader/<crypto.randomUUID()>` — one directory per download, git-ignored.
2. yt-dlp writes to `media.%(ext)s` inside it.
3. `findDownloadedFile` skips `.part` and `.ytdl` artifacts, tries preferred extensions in order, then falls back to a media-extension regex.
4. The file is read fully into a `Buffer` and returned with `fileName`, `mimetype` (from `mimeFor`), and `extension`.
5. `finally` removes the job directory with `rm(recursive, force)`.

Filenames are sanitized by `cleanFileName`: reserved characters and control codes stripped, whitespace collapsed, truncated to 80 chars. Spawned processes use `windowsHide: true` to avoid console flashes on Windows.

> The whole file is buffered in memory before sending, so the 64 MB cap is also a per-message heap cost. Under concurrent downloads this is the main memory pressure point.

### Interactive search flow

`createDownloadCommand({ name, desc, type, platform, usage, detailCard })` builds the shared downloader commands:

- **Direct link** → immediate download and send.
- **Search query** → `searchMedia` returns up to 5 results; the top result is posted as a thumbnail card captioned `Reply with 1 to download this video`, and state is stashed in a module-level `searchState` Map keyed by sender.
- A reply of `1`–`5` while state exists downloads that entry and **deletes the state immediately**, before any network work.
- State self-expires via `setTimeout(..., 2 * 60 * 1000)`.

Note the command-level `searchState` (2 min, no size cap) is separate from the dispatcher-level numeric routing in `messages.js`.

---

## Movie Engine

`src/engines/downloader/movie.engine.js` drives a multi-step conversational flow against a third-party HTTP API.

### State machine

Sessions are held in a module-level `Map` keyed by sender JID:

```
.search <query>
      │
      ▼
 SEARCH_SELECT ──.fetchMediaDetails(i)──┬─(downloads)──► QUALITY_SELECT
                                          │                  │
                                          └─(seasons)──► TV_SEASON_SELECT
                                                                │
                                              .fetchSeasonEpisodes(i)
                                                                ▼
                                                        TV_EPISODE_SELECT
                                                                │
                                              .fetchEpisodeDetails(i)
                                                                ▼
                                                        QUALITY_SELECT
                                                                │
                                                 .getFinalStream(i)
                                                                ▼
                                                          (deleted)
```

- `SESSION_TTL_MS = 300000` (5 minutes). Expiry is enforced lazily inside `hasActiveSession`, which sweeps stale entries on every call — there is no background timer.
- `searchTitle` deletes any prior session before starting, so a new search always resets cleanly.
- `getFinalStream` deletes the session **synchronously before returning**, preventing a completed session from being replayed.
- Entries with quality `SUB` (subtitle-only) are filtered out of every download list.
- `normalizeMediaType` classifies a hit as TV when its type string contains `tv`, `series`, or `show`, selecting `/tv/info` over `/info`.

The dispatcher consults `movieEngine.hasActiveSession(participant)` on every prefix-free numeric message, which is what allows bare-number navigation.

> This engine's API key and base URL are **hardcoded in source** at the top of the file. See [Security Notes](#security-notes).

---

## AI Layer

`src/commands/ai/_shared.js` centralizes the Pollinations transport.

### Chat transport

```
POST https://gen.pollinations.ai/v1/chat/completions
Headers: Content-Type: application/json
         Authorization: Bearer <POLLINATIONS_API_KEY | POLLINATIONS_KEY>   // only if present
Body:    { model, messages: [{role:'system',...}?, {role:'user',...}], temperature }
```

- The endpoint is OpenAI Chat Completions compatible, so any provider-specific command is a matter of swapping `model`.
- `temperature` defaults to `0.8` and is only included when `typeof temperature === 'number'`.
- The `Authorization` header is **omitted entirely** when no key is set, which is what allows keyless anonymous access to the free tier.
- `getReplyText` reads `data.choices[0].message.content.trim()`, falling back to `No response received.`

### Helpers

| Helper | Purpose |
|--------|---------|
| `getChatJid(sender, msg)` | `msg.key.remoteJid \|\| sender` — reply into the originating chat |
| `getPrompt(args)` | `args.join(' ').trim()` |
| `getPollinationsApiKey()` | First non-empty of `POLLINATIONS_API_KEY`, `POLLINATIONS_KEY` |
| `sendReaction(sock, sender, msg, text)` | Reaction, no-op without `msg.key` |
| `sendQuotedText(sock, sender, msg, text)` | Quoted reply |

### Image generation

`.imagine` bypasses the JSON transport entirely and builds a Pollinations image URL:

```
https://image.pollinations.ai/p/<encodeURIComponent(prompt)>?width=1024&height=1024&nologo=true
```

The resulting URL is sent as `{ image: { url } }`, so **generation happens on WhatsApp's servers fetching the URL**, not on the bot host — the bot never downloads or stores the image. Failures are caught locally and surfaced as `❌ CINEFy: Failed to generate image.`

---

## Logging

`src/lib/logger.js` implements the interface Baileys expects (the `logger` option) **without** a logging dependency:

| Method | Behavior |
|--------|----------|
| `info` | `console.log` with ISO timestamp — not persisted |
| `warn` | `console.warn` with ISO timestamp — not persisted |
| `error` | `console.error` **and** `fs.appendFileSync` to `logs/error.log` |
| `fatal` | `console.error` — not persisted |
| `debug` | **No-op** |
| `trace` | **No-op** |
| `child` | Returns the same logger (scoped children ignored) |

`formatLogArg` normalizes arguments: `Error` → `arg.stack || arg.message`, plain object → `JSON.stringify` inside a try/catch (circular structures degrade to `String(arg)`), everything else → `String(arg)`.

Consequences:

- **Only errors are durable.** `info`/`warn` exist solely in the console. There is no log rotation and no size cap — `logs/error.log` grows without bound.
- `debug` and `trace` are stubbed out. Baileys' verbose diagnostics are unavailable; enable them by adding `console.debug` to those two methods.
- `appendFileSync` is **synchronous**. Every error blocks the event loop on disk I/O. Fine for a low-rate bot, pathological under an error storm.
- The same `logger` object is injected into `makeWASocket`, so Baileys' internal errors land in `logs/error.log` alongside your own.
- The logger is passed to Baileys *instead of* configuring a real winston/pino instance — a genuine winston logger would give you levels, transports, and rotation for free.

---

## Project Layout

```
.
├── index.js                       # Entry: server → isPaired() → startBot()
├── config.js                      # Env parsing, path resolution
├── package.json
├── .env                           # Secrets (git-ignored)
├── logs/
│   └── error.log                  # Auto-created, errors only
├── auth/                          # Multi-file creds (git-ignored, session credential)
├── .cache/
│   └── downloader/<uuid>/         # Per-download job dirs (git-ignored)
└── src/
    ├── bot.js                     # Socket lifecycle, reconnect, pairing, status
    ├── api/
    │   ├── server.js              # Express app assembly
    │   └── routes.js              # /api/bot/* handlers
    ├── dashboard/
    │   ├── pages/index.html
    │   ├── styles.css
    │   └── utils/api.js           # Thin fetch wrapper over /api/bot
    ├── events/
    │   └── messages.js            # Command loader + dispatcher
    ├── commands/
    │   ├── general/               # menu, ping, alive, jid, getdp, vo, upload,
    │   │                          # compress, document, imagehost, longlink,
    │   │                          # reveal, movie, cinesubz, send, send_payload
    │   ├── downloader/            # ytmp4, ytmp3, ytm3, song, tiktok, fb, mediafire
    │   │                          # _downloadCommand.js (factory, not a command)
    │   ├── ai/                    # chat*, groq, mistral, openrouter, cerebra,
    │   │                          # kyrexi, meta, ai, imagine*
    │   │                          # _shared.js (transport, not a command)
    │   └── groups/                # kick, hidetag, welcome (exports `tagall`)
    ├── engines/downloader/
    │   ├── downloader.engine.js   # yt-dlp + ffmpeg orchestration
    │   └── movie.engine.js        # Conversational movie/TV session state machine
    ├── lib/
    │   ├── send.js                # sendText helper
    │   ├── logger.js              # Baileys-compatible logger
    │   └── fetch.js
    └── middleware/
        └── owneronly.js           # ownerOnly(sender, ownerNumber)
```

---

## Extending

### Add a command

Create `src/commands/<category>/<name>.js` — any depth works, the loader recurses.

```js
// src/commands/general/hello.js
export default {
  name: 'hello',
  aliases: ['hi'],
  desc: 'Greet the sender',
  category: 'general',
  run: async (sock, sender, args, msg) => {
    await sock.sendMessage(sender, { text: `Hello ${sender.split('@')[0]}` });
  }
};
```

No registration step. Restart the process (or `POST /api/bot/restart`).

### Gate a command by owner

The dispatcher supports this, but you must wire it yourself — nothing does today:

```js
import { ownerOnly } from '../../middleware/owneronly.js';

export default {
  name: 'danger',
  middleware: ownerOnly,        // called as middleware(participant, config.owner)
  run: async (sock, sender) => { /* ... */ }
};
```

`ownerOnly` strips non-digits from both sides before comparing.

> **`ownerOnly` fails open:** `if (!ownerNumber) return true`. If `OWNER_NUMBER` is unset or empty, *every* command guarded by this middleware is open to all users. It also compares only the number portion, so it does not distinguish a JID's device or LID suffix.

### Add an AI provider

Add a command that reuses the shared transport:

```js
// src/commands/ai/myprovider.js
import { callPollinationsChat, getPrompt, getReplyText, sendQuotedText } from './_shared.js';

export default {
  name: 'myprovider',
  aliases: ['mp'],
  desc: 'Query some model',
  category: 'ai',
  run: async (sock, sender, args, msg) => {
    const data = await callPollinationsChat({
      model: 'some-model-id',
      prompt: getPrompt(args),
      systemPrompt: 'You are a helpful assistant.'
    });
    await sendQuotedText(sock, sender, msg, getReplyText(data));
  }
};
```

### Add a downloader command

```js
// src/commands/downloader/ytwebm.js
import { createDownloadCommand } from './_downloadCommand.js';

export default createDownloadCommand({
  name: 'ytwebm',
  desc: 'Download a YouTube video as MP4',
  type: 'video',
  platform: 'youtube',
  usage: '.ytwebm <link | search query>'
});
```

---

## Deployment

### PM2

```bash
npm install -g pm2
pm2 start index.js --name "cinefy-bot"
pm2 save
pm2 startup
pm2 logs cinefy-bot
```

Use `--max-restart 10 --restart-delay 5000` if you want PM2 to supervise the process as well. Note the bot already self-reconnects internally, so PM2 restarts should be reserved for genuine crashes.

### systemd

```ini
[Unit]
Description=CINEFy MD WhatsApp Bot
After=network.target

[Service]
Type=simple
User=whatsapp
WorkingDirectory=/opt/cinefy-bot
ExecStart=/usr/bin/node index.js
Restart=always
RestartSec=10
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

### Docker

```dockerfile
FROM node:20-slim

# yt-dlp + ffmpeg are hard requirements for the downloader commands
RUN apt-get update && apt-get install -y --no-install-recommends \
      python3 python3-pip ffmpeg ca-certificates \
    && pip3 install --break-system-packages -U yt-dlp \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .

ENV NODE_ENV=production
EXPOSE 3000
VOLUME ["/app/auth"]

CMD ["node", "index.js"]
```

```bash
docker build -t cinefy-bot .
docker run -d \
  --name cinefy-bot \
  --restart unless-stopped \
  -p 127.0.0.1:3000:3000 \
  --env-file .env \
  -v "$(pwd)/auth:/app/auth" \
  cinefy-bot
```

Bind to `127.0.0.1` and reverse-proxy if you need external access — see the security notes.

### Keeping the session across deploys

`AUTH_DIR` must survive container replacement. Without a mounted volume, every deploy forces a re-pair. Back it up with the same care as any other credential.

---

## Security Notes

Review these before exposing the bot.

### 1. The dashboard is unauthenticated

Every `/api/bot/*` route is open, including `POST /logout`, which irreversibly destroys the paired session. There is no auth, no CSRF protection, and no rate limiting. Anyone who can reach the port can unpair the bot, force restarts into a reconnect loop, or read the connected JID.

**Mitigate:** bind to loopback (`127.0.0.1`) or front with an authenticating reverse proxy. Add middleware-level auth if the port must be exposed.

### 2. A credential is hardcoded in source

`src/engines/downloader/movie.engine.js` contains a live API key and base URL as module-level constants:

```js
const API_KEY = 'chama_api_...';
```

Unlike `.env` — which is git-ignored — **this file is tracked and will be committed**. Move it to an env var immediately:

```js
const API_KEY = process.env.CHAMA_API_KEY;
const BASE_URL = process.env.CHAMA_BASE_URL || 'https://chama-movie-api.koyeb.app/api/v1/movie/moviebox';
```

If this key has already been pushed, treat it as compromised and rotate it at the provider.

### 3. Live secrets are present in the working tree

The local `.env` contains populated values for Gemini, Mistral, Groq, OpenRouter, Cerebras, Kyrexi, and Pollinations. `.gitignore` correctly excludes `.env`, so verify with `git status` before your first commit that nothing sensitive is staged.

### 4. Owner-only commands are not actually gated

`src/middleware/owneronly.js` exists and the dispatcher honors `command.middleware`, but **no command uses it** — including `.send` and `.send_payload`, which are merely labelled `category: 'owner'`. `category` is never read by the dispatcher.

This means anyone in any chat where the bot is present can invoke `.send`, which repeatedly messages an arbitrary phone number, and `.send_payload`. Attach `middleware: ownerOnly` to both before deploying.

### 5. `ownerOnly` fails open

```js
if (!ownerNumber) return true;
```

An unset or empty `OWNER_NUMBER` disables every guard it is attached to. There is no warning at boot. Set the variable explicitly, and consider changing this to fail closed.

### 6. Bulk-messaging commands

`.send` loops `times` sends with `delay` ms between them to any number you supply, with no rate limit beyond your own delay argument and no allowlist. This is a spam primitive and a fast route to a WhatsApp ban. `.send_payload` is similar.

### 7. TLS verification is disabled for downloads

`--no-check-certificates` is passed to every yt-dlp invocation, removing protection against interception or a substituted source file.

### 8. `antiban` is off

`antiban: false` in the socket options disables Baileys' evasion heuristics. Combined with the bulk senders above, expect `401`/`403` disconnects and possible account restriction.

### 9. Unbounded log growth

`logs/error.log` is appended synchronously with no rotation or size cap. An error loop will fill the disk. Use a real logger with a rotating file transport.

### 10. Remote code loading scope

Every `.js` file under `src/commands` is imported at startup. That directory must be treated as trusted code — anything that can write there achieves code execution on the next restart.

---

## Credits

- [`amiudmodz`](https://www.npmjs.com/package/amiudmodz) — actively maintained Baileys fork used as the socket library
- [Baileys](https://github.com/WhiskeySockets/Baileys) — upstream WhatsApp Web library
- [UDMODZ](https://udmodz.site) — for baileys and kyrexi intergritation
- [AI Providers]


## License

ISC/MIT — see `package.json`.
