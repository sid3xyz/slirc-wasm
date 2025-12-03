# SLIRC WASM Bridge

A WebAssembly bridge for `slirc-proto`, enabling JavaScript environments to use the same IRC protocol parsing logic as the server and desktop client.

## Purpose

Eliminate parser inconsistencies between frontend and backend by compiling the Rust protocol implementation to WASM. The web client shares validation rules and message structures with the native client and server.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Browser                               │
│  ┌──────────────────┐         ┌──────────────────────────┐  │
│  │   JavaScript     │  JSON   │     WASM Module          │  │
│  │  - WebSocket IO  │◄───────►│  - parse(string)         │  │
│  │  - UI / DOM      │         │  - build(object)         │  │
│  │  - User Input    │         │  - slirc-proto structs   │  │
│  └────────┬─────────┘         └──────────────────────────┘  │
│           │                                                  │
└───────────┼──────────────────────────────────────────────────┘
            │ WebSocket
            ▼
     ┌──────────────┐
     │  IRC Server  │
     └──────────────┘
```

### Data Flow

| Direction | Flow |
|-----------|------|
| Incoming  | Server → WebSocket → JS → `parse(raw)` → WASM → JSON object |
| Outgoing  | User → JS → `build(obj)` → WASM → raw string → WebSocket → Server |

## API

```typescript
// Core
export function parse(input: string): IrcMessage;
export function build(msg: IrcMessage): string;
export function version(): string;

// Helpers (planned)
export function privmsg(target: string, text: string): string;
export function join(channel: string): string;
```

## Implementation

### Phase 1: Configuration

```toml
# slirc-wasm/Cargo.toml
[package]
name = "slirc-wasm"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
serde = { version = "1.0", features = ["derive"] }
serde-wasm-bindgen = "0.6"
console_error_panic_hook = "0.1"

[dependencies.slirc-proto]
path = "../slirc-proto"
default-features = false
features = ["serde"]

[profile.release]
opt-level = "s"
lto = true
codegen-units = 1
strip = true
panic = "abort"
```

### Phase 2: Core Implementation

```rust
// src/lib.rs
use wasm_bindgen::prelude::*;
use slirc_proto::Message;

#[wasm_bindgen(start)]
pub fn init() {
    console_error_panic_hook::set_once();
}

#[wasm_bindgen]
pub fn parse(input: &str) -> Result<JsValue, JsValue> {
    let msg: Message = input.parse()
        .map_err(|e| JsValue::from_str(&e.to_string()))?;
    serde_wasm_bindgen::to_value(&msg)
        .map_err(|e| JsValue::from_str(&e.to_string()))
}

#[wasm_bindgen]
pub fn build(val: JsValue) -> Result<String, JsValue> {
    let msg: Message = serde_wasm_bindgen::from_value(val)
        .map_err(|e| JsValue::from_str(&e.to_string()))?;
    Ok(msg.to_string())
}

#[wasm_bindgen]
pub fn version() -> String {
    env!("CARGO_PKG_VERSION").to_string()
}
```

### Phase 3: TypeScript Types

Option A: Manual `.d.ts` (simpler, no proto changes):
```typescript
// pkg/slirc_wasm.d.ts (manual addition)
export interface IrcMessage {
  tags?: Record<string, string>;
  prefix?: { Server: string } | { Nick: [string, string?, string?] };
  command: Record<string, any>;
}
```

Option B: `tsify` integration (requires proto changes):
- Add `wasm` feature to `slirc-proto`
- Derive `Tsify` on `Message`, `Command`, `Prefix`

### Phase 4: Frontend Integration

```javascript
import init, { parse, build } from './pkg/slirc_wasm.js';

await init();

socket.onmessage = (e) => {
  const msg = parse(e.data);
  handleMessage(msg);
};

function send(command) {
  socket.send(build(command));
}
```

## Build & Test

```bash
# Build
wasm-pack build --target web --release

# Test
cargo test
wasm-pack test --headless --firefox

# Size check (target: <100KB gzipped)
ls -lh pkg/*.wasm
gzip -c pkg/slirc_wasm_bg.wasm | wc -c
```

## Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Parsing location | WASM | Single source of truth |
| Serialization | `serde-wasm-bindgen` | Low overhead, native JS types |
| TypeScript | Manual `.d.ts` initially | Avoids proto modifications |
| WebSocket | JS-side | Better browser API integration |

## Files

```
slirc-wasm/
├── Cargo.toml
├── src/
│   └── lib.rs
├── tests/
│   └── web.rs
└── README.md
```
