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

- [x] Create crate structure
- [ ] Implement `parse_message` wrapper around `slirc_proto::Message`
- [ ] Implement `build_message` wrapper
- [ ] Add serialization support (Serde) for JS objects
- [ ] Add unit tests ensuring JS compatibility
