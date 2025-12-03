# Web Client Implementation Plan

This document outlines the strategy for implementing a functional web client for `slirc.net` using a WASM-based protocol bridge.

## Architecture

The web client uses a hybrid architecture:
- **UI**: HTML/CSS/JS (existing "cyberpunk" terminal theme).
- **Protocol**: Rust compiled to WASM (via `slirc-wasm` crate).
- **Transport**: WebSocket (`wss://`) connecting to `slircd-ng`.

This approach ensures that the web client uses the exact same protocol parsing logic (`slirc-proto`) as the server and desktop client, guaranteeing consistency and correctness.

## Components

### 1. Backend (`slircd-ng`)
- **Goal**: Expose a WebSocket endpoint for web clients.
- **Tasks**:
    - Enable `[websocket]` listener in `config.toml`.
    - Verify WebSocket framing implementation in `slircd-ng`.
    - Configure TLS/SSL (likely via reverse proxy or Cloudflare).

### 2. Protocol Bridge (`slirc-wasm`)
- **Goal**: Expose `slirc-proto` functionality to JavaScript.
- **Location**: `/home/straylight/slirc-wasm`
- **Tasks**:
    - Implement `parse_message(input: &str) -> JsValue`
    - Implement `build_message(command: &str, params: Vec<&str>) -> String`
    - Build pipeline using `wasm-pack`.

### 3. Frontend (`slirc.net/www/client`)
- **Goal**: Connect the UI to the backend.
- **Tasks**:
    - Replace mock data with real WebSocket connection.
    - Import `slirc-wasm` for message parsing.
    - Map IRC events (JOIN, PRIVMSG, etc.) to UI state updates.

## Implementation Roadmap

### Phase 1: Foundation
- [x] Create `slirc-wasm` crate structure.
- [ ] Implement basic parsing in `slirc-wasm`.
- [ ] Verify `slircd-ng` WebSocket support.

### Phase 2: Integration
- [ ] Build `slirc-wasm` and copy artifacts to `www/client/pkg/`.
- [ ] Write JS WebSocket wrapper in `www/client/`.
- [ ] Connect to local `slircd-ng` instance for testing.

### Phase 3: Deployment
- [ ] Automate `wasm-pack` build in deployment pipeline.
- [ ] Deploy to `slirc.net`.
