# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

TACO! GATO! CHAOS! — a P2P party reflex game (2–6 players) shipped as a static PWA for GitHub Pages. No backend, no build step, no package manager. UI text and code comments are in Portuguese (pt-BR). `SPEC.md` is the design spec; read it for game rules, the localStorage profile schema, bot failover and the reconnect flow.

## Running and testing

- Serve the folder over HTTP (the service worker and PeerJS need it), e.g. `npx serve .` or `python -m http.server 8000`, then open `index.html`.
- Multiplayer locally: open two tabs. The same browser profile is supported. `tabSessionId` tells tabs of one `userId` apart, and the host remaps a same-machine second tab to a distinct user.
- Invite links use `#sala=<CODE>`. On load, the page auto-switches to the lobby and joins.
- Tests: open `test_e2e.html` in a browser. Results render on the page and in the console (`PASS:`/`FAIL:`). There is no CLI runner.
- **Important:** `test_e2e.html` does not load `index.html`. It holds its own copies of the host/client message handlers, `getTableOverlayHtml` and a `MockConnection` pair that links two in-memory peers. Changes to game or protocol logic in `index.html` must be mirrored by hand in the test file, or the tests will pass against stale logic.
- After changing cached assets, bump `CACHE_NAME` in `sw.js` (currently `taco-chaos-v9`). Navigations are network-first, but other assets are cache-first and keep old versions until the bump.

## Architecture

Nearly everything lives in `index.html` (~2400 lines): Tailwind via CDN (theme tokens in the inline `tailwind.config`), PeerJS 1.5.4 from unpkg, markup, and one inline script. The script is split by numbered `// ====` banner sections: user account, Web Audio SFX (synth tones, no audio files), card data (`CARDS_DATA` with inline SVG, `SEQUENCE`, `CHAOS_MODIFIERS`, `DEFEAT_MEMES`), game state, P2P room setup, message protocol, lobby, game engine, stickers, HUD rendering, and init.

### Host-authoritative star topology
- The room creator is the host. Its PeerJS id is `taco-p2p-<roomcode lowercase>`, so clients connect from the code alone. Signaling uses the public PeerJS server. Data then flows over the WebRTC data channel.
- Single global `gameState`. `isMultiplayer`/`isHost` select the role. `slots[]` holds players (`userId`, `isBot`, `isHost`, `cardsCount`, `conn`). `mySlotIndex` is the local player's slot.
- Only the host mutates game state and runs the `execute*` functions (`executePlayCard`, `executeRegisterSlap`, `executeFalseSlap`, `executeResolvePileToLoser`), bot timers (`scheduleBotTurn`, slap timers in `activateTriggerEvent`) and the win check. Clients send intents and apply the broadcast results in `handleClientReceivedMessage`.
- Solo mode (`startSoloGame`) runs the same engine locally with the player acting as host. Bots are plain slots with `isBot: true`, driven by host timers.
- `broadcastToClients` sends to every non-bot slot connection. `getSanitizedSlots`/`getPublicGameState` strip `conn` objects before serializing. Never send raw `slots` over the wire.
- Disconnect: `handleClientDisconnection` turns the slot into a bot. Reconnect: the client sends `JOIN_OR_RECONNECT`, the host matches by `userId`, clears `isBot`, rebinds `conn` and replies with a state snapshot (`applySyncedState` on the client).

### Message types actually used
The implementation diverges from the table in `SPEC.md` section 3. Trust the code:
- Client to host: `JOIN_OR_RECONNECT` (from an already-seated connection it only updates name/avatar), `PLAYER_PLAY_CARD`, `PLAYER_SLAP`, `SEND_STICKER`, `PONG`
- Host to clients: `JOIN_ACCEPTED`, `RECONNECT_SUCCESS`, `ROOM_FULL`, `GAME_IN_PROGRESS`, `ROOM_CLOSED`, `ROOM_UPDATE`, `GAME_STARTED`, `CARD_PLAYED` (carries `counts` and `trigger`), `SLAP_STAMP` (provisional), `SLAP_RESULT`, `ROUND_RESOLVED`, `ROUND_UNLOCKED`, `GAME_OVER`, `STICKER_BROADCAST`, `PLAYER_DISCONNECTED`, `PLAYER_RECONNECTED`, `PING`
- The host never trusts a `slotIndex` from the client: it derives the slot from the connection. `broadcastToClients` adds the recipient's `yourSlotIndex` to every packet, so clients re-locate themselves when slots shift.
- Cards, modifiers and memes travel as ids (`cardId`, `modifierId`, `memeIndex`) and are looked up in local tables. Never send HTML over the wire; escape names/avatars with `escapeHtml` before `innerHTML`.
- Match timers go through `gameTimeout` (cancelled by `clearAllTimers`, ignored after `gameId` changes). `teardownSession` must run before starting solo, creating or joining a room.
- Heartbeat: host `PING` every 2s carries `t` (echoed in `PONG` to measure per-slot `rtt`) and the `pings` table shown in the opponents bar; a client silent for 7s becomes a bot. Clients auto-reconnect up to 5 times with backoff.
- Slap fairness: the slap order is decided by reaction time (`triggerShownAt` to tap, measured on each device), not by arrival at the host. Clients send `PLAYER_SLAP { reactionMs }`. `computeReactionMs` clamps it to `[elapsed - rtt - jitter, elapsed]`. When everyone has slapped, `computeSlapRanking` broadcasts `SLAP_RESULT`. The slap fires on `pointerdown`.

### Persistence
The profile is stored in `localStorage` under key `taco_chaos_user`. See `initUserAccount` and `saveUserToStorage`.
