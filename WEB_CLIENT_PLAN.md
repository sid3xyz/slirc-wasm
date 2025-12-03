# SLIRC WASM Bridge

This crate provides a WebAssembly bridge for the `slirc-proto` library, allowing JavaScript environments (like the `slirc.net` web client) to use the exact same IRC protocol parsing logic as the server and desktop client.

## Purpose

To eliminate parser inconsistencies between the frontend and backend by compiling the Rust protocol implementation to WASM.

## Research Findings

### Existing Serde Support

**Good news**: `slirc-proto` already has Serde support behind a feature flag.

The following types already derive `Serialize`/`Deserialize` when `features = ["serde"]`:

| Type       | Location           | Notes                        |
| ---------- | ------------------ | ---------------------------- |
| `Message`  | `message/types.rs` | Top-level message struct     |
| `Command`  | `command/types.rs` | Large enum with ~80 variants |
| `Prefix`   | `prefix/types.rs`  | Server or Nickname variant   |
| `Response` | `response/mod.rs`  | Numeric reply codes          |
| `Tag`      | `message/types.rs` | IRCv3 message tags           |

### JSON Serialization Format

Serde's default enum serialization will produce **externally tagged** JSON:

```json
// Command::PRIVMSG("#channel", "hello")
{ "PRIVMSG": ["#channel", "hello"] }

// Prefix::Nickname("nick", "user", "host")
{ "Nickname": ["nick", "user", "host"] }

// Prefix::ServerName("irc.example.com")
{ "ServerName": "irc.example.com" }
```

This is usable but slightly awkward for JS. We may want to provide helper functions or consider `#[serde(tag = "type")]` in the future.

### Reference Implementation

Studied `matrix-org/matrix-rust-sdk` (Apache-2.0 license). Key patterns:

1. **Use `serde-wasm-bindgen`** for struct conversion (not manual field access).
2. **No tokio in WASM**: Use `wasm_bindgen_futures::spawn_local` if async needed.
3. **Error conversion**: Return `Result<JsValue, JsValue>` for JS try/catch compatibility.

## API Goals

This crate will expose the following functionality to JavaScript via `wasm-bindgen`:

### Parsing

```rust
#[wasm_bindgen]
pub fn parse(input: &str) -> Result<JsValue, JsValue> {
    let msg: Message = input.parse().map_err(|e| e.to_string())?;
    Ok(serde_wasm_bindgen::to_value(&msg)?)
}
```

### Construction

```rust
#[wasm_bindgen]
pub fn build(val: JsValue) -> Result<String, JsValue> {
    let msg: Message = serde_wasm_bindgen::from_value(val)?;
    Ok(msg.to_string())
}
```

## TypeScript Interface (Expected Output)

```typescript
interface IrcMessage {
  tags?: Array<{ key: string; value?: string }>;
  prefix?: { ServerName: string } | { Nickname: [string, string, string] };
  command: { [commandName: string]: any[] };
}

// Example parsed message
const msg = parse(":nick!user@host PRIVMSG #channel :Hello!");
// {
//   prefix: { Nickname: ["nick", "user", "host"] },
//   command: { PRIVMSG: ["#channel", "Hello!"] }
// }
```

## Build Instructions

```bash
# Install wasm-pack if not present
cargo install wasm-pack

# Build for web target
wasm-pack build --target web --release

# Output in ./pkg/
```

## Implementation Roadmap

### Phase 1: Project Configuration

- [x] Initialize `slirc-wasm` crate
- [x] Add to workspace `Cargo.toml`
- [ ] Update `slirc-wasm/Cargo.toml`:
  - Enable `serde` feature on `slirc-proto`
  - Add `console_error_panic_hook` for debugging
  - Configure `[profile.release]` for WASM size optimization
- [ ] Create `.cargo/config.toml` for WASM-specific settings (optional)

### Phase 2: Core Bridge Implementation

- [ ] Implement `parse(input: &str) -> Result<JsValue, JsValue>`
  - Parse raw IRC line to `Message`
  - Serialize to JS object via `serde_wasm_bindgen::to_value`
  - Return parse errors as strings
- [ ] Implement `build(val: JsValue) -> Result<String, JsValue>`
  - Deserialize JS object to `Message`
  - Serialize to IRC string via `Message::to_string()`
  - Return validation errors as strings
- [ ] Add `version()` function (already present as placeholder)

### Phase 3: Developer Experience

- [ ] Generate TypeScript definitions
  - Option A: Use `tsify` crate for auto-generation
  - Option B: Write manual `.d.ts` file
- [ ] Add helper functions for common commands:
  - `privmsg(target: &str, text: &str) -> String`
  - `join(channel: &str) -> String`
  - `nick(nickname: &str) -> String`
- [ ] Document JSON structure of each command variant

### Phase 4: Testing

- [ ] Unit tests in Rust (`cargo test`)
- [ ] WASM tests (`wasm-bindgen-test` in headless browser)
- [ ] Round-trip tests: `parse(build(x)) == x`
- [ ] Fuzz testing for parse function (optional)

### Phase 5: CI/CD & Release

- [ ] GitHub Actions workflow:
  - Run `cargo test`
  - Run `wasm-pack test --headless --chrome`
  - Build release artifacts
- [ ] Publish to npm (optional, can also just copy `pkg/` to `slirc-net`)
- [ ] Add usage documentation for frontend integration

## Dependencies

```toml
[dependencies]
wasm-bindgen = "0.2"
serde-wasm-bindgen = "0.6"
console_error_panic_hook = "0.1"
slirc-proto = { path = "../slirc-proto", default-features = false, features = ["serde"] }

[dev-dependencies]
wasm-bindgen-test = "0.3"
```

## Size Optimization

Add to `Cargo.toml`:

```toml
[profile.release]
opt-level = "s"      # Optimize for size
lto = true           # Link-time optimization
codegen-units = 1    # Single codegen unit for better optimization
```

Expected bundle size: ~100-300KB (gzipped: ~30-80KB)

## Additional Research Findings

### TypeScript Generation with `tsify`

The `tsify` crate (Apache-2.0/MIT) automatically generates TypeScript definitions from Rust structs. Key features:

- Integrates with `serde-wasm-bindgen` via `features = ["js"]`
- Derives `IntoWasmAbi`/`FromWasmAbi` so no manual `from_value`/`to_value` calls
- Supports Serde attributes (`rename`, `tag`, `untagged`, etc.)

Example:
```rust
use tsify::Tsify;

#[derive(Tsify, Serialize, Deserialize)]
#[tsify(into_wasm_abi, from_wasm_abi)]
pub struct Point { x: i32, y: i32 }
```

Generates:
```typescript
export interface Point { x: number; y: number; }
```

**Decision**: Use `tsify` for Phase 3 TypeScript generation.

### `serde-wasm-bindgen` Configuration Options

The serializer can be customized:

- `serialize_missing_as_null(true)`: Serialize `None` as `null` instead of `undefined`
- `serialize_maps_as_objects(true)`: Serialize `HashMap` as plain objects instead of `Map`
- `Serializer::json_compatible()`: Preset for JSON-like output

**Recommendation**: Use `Serializer::json_compatible()` for more intuitive JS objects.

### Existing IRC Web Client Architectures

Researched how other web IRC clients handle this:

| Project | Architecture | Protocol Layer | License |
|---------|-------------|----------------|---------|
| **The Lounge** | Node.js server + Vue frontend | JS parser (irc-framework) | MIT |
| **Kiwi IRC** | Static files + WebSocket proxy | JS parser | Apache-2.0 |
| **Gamja** | Static files + WebSocket | JS parser | AGPL-3.0 |

**Key insight**: All existing clients use JavaScript parsers. Our WASM approach is novel in the IRC space.

### WebSocket Gateway Options

If `slircd-ng` native WebSocket support is incomplete, we can use a gateway:

1. **webircgateway** (kiwiirc): Go-based proxy, well-tested, Apache-2.0
   - Handles WEBIRC, encoding, multiple transports
   - Production-ready, used by many networks

2. **Native WebSocket in slircd-ng**: Already partially implemented
   - `websocket.rs` in `slirc-proto` has `WebSocketConfig`
   - `config.toml` has commented `[websocket]` section

**Recommendation**: First verify `slircd-ng` native WebSocket works. Fall back to `webircgateway` if needed.

### Browser Compatibility Notes

From `wasm-bindgen` docs:

- All modern browsers support WASM
- IE11 does NOT support WASM (acceptable loss)
- Mobile browsers fully supported
- WebSocket support is universal in modern browsers

### Security Considerations

From `webircgateway` best practices:

1. **Origin validation**: Only allow connections from trusted domains
2. **HTTPS required**: Browsers block mixed content (WSS from HTTPS pages)
3. **WEBIRC**: Preserve real user IPs for bans to work correctly
4. **Rate limiting**: Web clients can be easily scripted for abuse

## Architecture Decision Record

### ADR-001: Use WASM for Protocol Parsing

**Context**: Need IRC protocol parsing in browser for web client.

**Decision**: Compile `slirc-proto` to WASM rather than write a JS parser.

**Rationale**:
1. Single source of truth for parsing logic
2. Guarantees consistency between server and client
3. Leverages existing tested code
4. Zero maintenance of duplicate parser

**Trade-offs**:
- Slightly larger bundle size (~100KB vs ~20KB for minimal JS parser)
- Build complexity (requires `wasm-pack`)
- Novel approach with less community precedent

**Status**: Accepted

### ADR-002: Use `serde-wasm-bindgen` for Serialization

**Context**: Need to convert Rust structs to JavaScript objects.

**Decision**: Use `serde-wasm-bindgen` instead of manual `wasm-bindgen` getters.

**Rationale**:
1. Official recommended approach
2. Much smaller code size than JSON-based alternatives
3. Native JavaScript types (Map, Array, etc.)
4. Integrates with `tsify` for TypeScript

**Status**: Accepted

### ADR-003: TypeScript Generation with `tsify`

**Context**: Frontend developers need type definitions.

**Decision**: Use `tsify` crate with `features = ["js"]`.

**Rationale**:
1. Automatic generation from Rust types
2. Stays in sync with Rust changes
3. Supports Serde attributes for customization

**Status**: Proposed (pending implementation)
