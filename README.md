# emulatorSHARE

Let a room full of people play one machine at the same time. A host streams its
screen + audio to the relay, and every connected viewer watches, chats, and
presses buttons into a shared input pile that gets replayed on that machine.
Viewers never run the game - they get pixels.

There are two kinds of host:

- **Full host (recommended)** - streams a real Linux desktop. Games are plain
  Linux binaries listed in `games.json`; viewers vote on which one to launch.
- **Headless-Chromium host (deprecated)** - legacy mode that ran WASM emulators
  (like an Emscripten build of an emulator) inside headless Chromium, one per
  console. It still works but is no longer the supported way to host.

```
 host (full Linux desktop) ->ws /host-> [ relay server.js ] -ws /stream-> viewers
 host <- merged input/controller ------------ relay <----- buttons/mouse/chat
```

Consoles are **dynamic**: nothing is hardcoded. When a host boots, it connects
to `/host?token=...&console=<key>`, sends a `register` frame (name, image,
category, description), and the relay creates or updates that console in its
SQLite database. Viewers discover it in the grid and connect via
`/stream?token=...&console=<key>`.

## Stream a desktop into it (full host, recommended)

The full host is a **Linux** machine that runs a real virtual desktop and
encodes it to the stream. It needs a handful of native tools:

```bash
sudo apt install xvfb xfwm4 pulseaudio ffmpeg xdotool
```

Then point it at your relay (which can be any machine on your network):

```bash
npm run host-full -- --url ws://192.168.1.20:8090 \
    --token <EMULATOR_HOST_TOKEN> \
    --console playground --name "Linux Playground" --games games.json
```

What it does:

- starts `Xvfb` (the virtual display viewers see), `xfwm4` (a window manager
  so launched games get decorated windows), and `pulseaudio` (game audio +
  mouse/keyboard replay) on the display,
- encodes that desktop with `ffmpeg` (VP8 or H.264 video, Opus audio) and
  streams it to your relay's `/host` WebSocket,
- registers the console and shows it in the viewer grid the moment it connects.

Key options (`npm run host-full -- --help` for all):

- `--games <path>` - `games.json` describing launchable games (default
  `games.json` at the repo root; see `games.example.json`),
- `--video-codec <h264|vp8>` - encoder for the video stream (default `h264`),
- `--resx --resy` - virtual display resolution (default = `--w` x `--h`),
- `--w --h --fps --bitrate` - pixel size / frame rate / bitrate of the stream,
- `--keys <all|none|list>` - which keys viewers may send, e.g.
  `w,a,s,d,space` to allow only an allowlist,
- `--motd` - message posted to chat when someone joins,
- `--check` - verify all required binaries + `games.json` and exit.

### games.json and voting

Copy `games.example.json` to `games.json` and list one entry per launchable
game: `key` (unique id), `name` (shown to viewers), `command` (argv, first item
on PATH or absolute path), plus optional `cwd`/`env`:

```json
{
  "games": [
    { "key": "srb2", "name": "Sonic Robo Blast 2", "command": ["/usr/bin/srb2", "-opengl"] },
    { "key": "dosbox", "name": "DOSBox", "command": ["/usr/bin/dosbox"], "env": { "SDL_FULLSCREEN": "0" } }
  ]
}
```

Any viewer can propose launching a game; the other viewers vote
yes/no against the `needed` threshold. When a vote passes the relay replies
`launch` and the full host starts that game's command on the virtual display.
Viewer input is replayed with `xdotool`.

> **Note:** The full host is Linux-only. Every tool it needs (Xvfb, xfwm4,
> pulseaudio, ffmpeg, xdotool) is a native Linux binary.

## Legacy: headless-Chromium host (deprecated)

This is the original mode. Instead of a real desktop, each console is an
**Emscripten/SDL emulator** (e.g. `sm64.js`/`sm64.wasm`) running inside
headless Chromium; `host/` (`host.js` + `serve-host.js`) captures that canvas
and audio with WebCodecs and `bin/launch-host.js` boots it:

```bash
npm run host-console -- --url ws://192.168.1.20:8090 \
    --token <EMULATOR_HOST_TOKEN> \
    --console mario64 --dir consoles/mario64 \
    --name "Super Mario 64" --category "Nintendo 64"
```

It still runs, but it is **deprecated** - new setups should use the full host
(`npm run host-full`), which hosts arbitrary native games instead of one
WASM emulator per console. Some caveats that motivated the deprecation:
Windows hosting was buggy (the software-GL emulator stutters whenever other
GPU-heavy Chromium windows are foregrounded) and consoles were folders of
hand-authored `index.html` files rather than plain `games.json` entries.

## Run the relay (the server)

```bash
npm install
cp .env.example .env            # set EMULATOR_HOST_TOKEN + EMULATOR_JWT_SECRET
node server.js                  # relay on :8090, serves public/ + API
```

Open http://localhost:8090, register, and log in. For the server to accept
remote hosts it must be reachable over the network (it already binds
`0.0.0.0`) and you must share `EMULATOR_HOST_TOKEN`.

## Stack

- **server.js** - the relay (`ws`), auth (username/password via scrypt), signed
  viewer tokens, and SQLite (`better-sqlite3`) for users, sessions, and the
  console registry. Per-console: viewer fan-out, key/mouse merge (anarchy or
  majority democracy), keyframe cache, watchdog, chat, votes, and a live roster.
- **bin/launch-full-host.js** - the full host: Xvfb + xfwm4 + pulseaudio,
  `ffmpeg` capture (VP8/H.264 + Opus), xdotool input replay, and the
  `games.json` vote-to-launch flow.
- **bin/launch-host.js** + **host/** - the legacy headless-Chromium host
  (WebCodecs capture, `serve-host.js` loopback static server). Deprecated.
- **public/** - the viewer web app (login/register, console grid, stream viewer
  with WebCodecs decode, on-screen + keyboard controls, chat, player list).
- **consoles/** - legacy console definitions for the deprecated Chromium host
  (`index.html` host page + `bin/`); `demo/` is a self-contained canvas console
  that tests the whole pipeline without a real ROM.

## Protocol

Media frames: `[kind:u8][timestamp:f64][payload]`, kinds `2`=video-keyframe,
`3`=video-delta, `5`=audio-chunk. Config `vconfig`/`aconfig`, roster, chat,
input, vote, mode, held, keyframe, reload, and launch are JSON control messages
on the same sockets. See `server.js` and `test/protocol.test.js`.

## Test

```bash
npm test
```