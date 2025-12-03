# SLIRC WASM Bridge

This crate provides a WebAssembly bridge for the `slirc-proto` library, allowing JavaScript environments (like the `slirc.net` web client) to use the exact same IRC protocol parsing logic as the server and desktop client.

## Purpose

To eliminate parser inconsistencies between the frontend and backend by compiling the Rust protocol implementation to WASM.

## API Goals

This crate will expose the following functionality to JavaScript via `wasm-bindgen`:

### Parsing
```rust
// Input: Raw IRC line from WebSocket
// Output: Structured JS object
fn parse_message(input: &str) -> Result<JsValue, JsValue>
```

### Construction
```rust
// Input: Command and parameters
// Output: Raw IRC line to send over WebSocket
fn build_message(command: &str, params: Vec<&str>) -> String
```

## Build Instructions

This project uses `wasm-pack` to generate the WASM binary and JavaScript glue code.

```bash
wasm-pack build --target web
```

## Implementation Roadmap

### Phase 1: Project Configuration & Dependencies
- [x] Initialize `slirc-wasm` crate.
- [ ] Update `Cargo.toml` to enable `serde` feature in `slirc-proto`.
- [ ] Configure `wasm-pack` build profile for size optimization.
- [ ] Set up `console_error_panic_hook` for better debugging in browser.

### Phase 2: Core Protocol Bridge
- [ ] **Parsing**: Implement `parse_message(input: &str) -> Result<JsValue, JsValue>`.
    - Should accept a raw IRC line.
    - Should return a JSON-serializable object (via `serde-wasm-bindgen`).
    - Should handle parsing errors gracefully and return meaningful error strings.
- [ ] **Construction**: Implement `build_message(command: &str, params: Vec<String>, tags: Option<JsValue>) -> Result<String, JsValue>`.
    - Should accept command, parameters, and optional tags.
    - Should validate inputs using `slirc-proto` validation logic.
    - Should return the formatted IRC string.

### Phase 3: Data Structure Exposure
- [ ] Define TypeScript definitions (via `wasm-bindgen` or manual `.d.ts`) for the Message object structure.
- [ ] Ensure `slirc-proto` types (Message, Prefix, Tags) serialize to intuitive JSON shapes.
    - *Note*: May need to create wrapper structs in `slirc-wasm` if the default `slirc-proto` serialization is too Rust-centric.

### Phase 4: Testing & Validation
- [ ] **Unit Tests**: Write Rust unit tests for the bridge functions.
- [ ] **WASM Tests**: Configure `wasm-bindgen-test` to run tests in a headless browser/Node.js.
- [ ] **Round-trip Verification**: Ensure `parse(build(cmd, params))` returns the original data.

### Phase 5: CI/CD & Release
- [ ] Create GitHub Actions workflow to run `cargo test` and `wasm-pack test`.
- [ ] Add release step to build artifacts (`pkg/` folder) and publish to GitHub Releases or NPM.
- [ ] Document usage instructions for the frontend team (how to import and use).
