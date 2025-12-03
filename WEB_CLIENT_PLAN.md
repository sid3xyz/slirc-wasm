# SLIRC WASM Bridge

This crate provides a WebAssembly bridge for the `slirc-proto` library, allowing JavaScript environments (like the `slirc.net` web client) to use the exact same IRC protocol parsing logic as the server and desktop client.

## Purpose

To eliminate parser inconsistencies between the frontend and backend by compiling the Rust protocol implementation to WASM.

## Research Findings

### Existing Serde Support

**Good news**: `slirc-proto` already has Serde support behind a feature flag.

The following types already derive `Serialize`/`Deserialize` when `features = ["serde"]`:

| Type | Location | Notes |
|------|----------|-------|
| `Message` | `message/types.rs` | Top-level message struct |
| `Command` | `command/types.rs` | Large enum with ~80 variants |
| `Prefix` | `prefix/types.rs` | Server or Nickname variant |
| `Response` | `response/mod.rs` | Numeric reply codes |
| `Tag` | `message/types.rs` | IRCv3 message tags |

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
