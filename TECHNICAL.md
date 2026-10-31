kmatrix Technical Documentation

This document covers the architectural rationale, technical constraints, security model, and build procedures for kmatrix.

Architectural Rationale

kmatrix intentionally separates the network and cryptographic handling from the presentation layer. KOReader’s native Lua environment presents several fundamental constraints:

Synchronous Execution: KOReader lacks async HTTP capabilities. Long-polling Matrix /sync endpoints directly in Lua blocks the primary UI thread, freezing touch rendering and page turns.

Insecure TLS Handling: The bundled LuaSec library in standard KOReader builds defaults to verify = "none", leaving HTTP access tokens vulnerable to MITM attacks.

Socket Restrictions: The bundled LuaSocket exports only luaopen_socket_score, omitting socket.unix support. Loopback TCP over local host interfaces is required.

Memory Overhead: Standard Lua JSON decoders materialize full payload trees into memory. Large Matrix sync payloads quickly exhaust e-reader RAM limits.

The kmatrixd Rust daemon resolves these limitations by executing long-running background tasks independently and delivering pre-parsed, light-weight JSON payloads to the Lua frontend.

Measured Performance & Behavior

Zero Generic Deserialization: The Rust sync pipeline deserializes incoming streams directly into strictly typed structures rather than generic JSON ASTs.

Heap Footprint: Peak memory usage remains around ~4 MiB regardless of account size (compared to ~1.4 GB peak when parsing equivalent payloads into a generic serde_json::Value tree).

Crypto Benchmark: Olm/Megolm operations execute in roughly 0.6 ms per message on target ARM hardware, running at ~3 MB RSS.

Binary Portability: A single statically linked armv7-unknown-linux-musleabihf binary targets Kindle PW5, reMarkable 2, and armv7 Kobo devices interchangeably.

Data at Rest & Encryption Model

USB Storage Vulnerability

On Kindle hardware, the standard KOReader directory resides on /mnt/us (the USB mass storage partition). Connecting an unencrypted e-reader to a computer grants complete filesystem access without authentication.

Key Isolation

kmatrixd protects message contents, access tokens, and private keys by maintaining an encrypted SQLite store:

Master Key Path: /var/local/kmatrix/store.key (mode 0600, stored on /dev/mmcblk0p9, an isolated system partition inaccessible via standard USB connection).

Encryption Implementation: Powered by vodozemac's Cipher. Values are encrypted with fresh 16-byte salts and derived HKDF keys (AES-256 + 32-byte HMAC validation).

Encrypted Fields: Message bodies, retained ciphertexts, room names, previews, access tokens, key backup keys, and Olm pickle keys.

Unencrypted Metadata: Event IDs, room IDs, sender IDs, and timestamps remain readable.

Note: Override key storage location using --key-dir <path>, or disable encryption explicitly via --no-encryption. To reset a corrupted database or missing master key, execute with --reset-store.

Build System & Cross-Compilation

Target e-readers run a 32-bit hard-float ARM environment on glibc ~2.20 with kernel 4.9. Dynamically linked binaries fail due to glibc version mismatches (GLIBC_2.xx not found). Fully static compilation under musl is mandatory.

Nix Flake Targets

# Build complete distribution bundle (kmatrixd + kmatrix.koplugin)
nix build .#

# Build compressed release tarball (.tar.gz)
nix build .#tarball

# Build static armv7 daemon binary only
nix build .#armv7

# Host build (runs native test suite)
nix build .#kmatrixd

# Execute host daemon locally
nix run .# -- --data-dir ./scratch



armv7 cross-compiles using pkgsCross.armv7l-hf-multiplatform.pkgsMusl with target flag -C target-feature=+crt-static.

Development Shell (cargo-zigbuild)

For faster iterative builds without full Nix derivations:

nix develop

# Package release artifact via script
./scripts/package.sh

# Manual cross-compilation
cargo zigbuild --release --target armv7-unknown-linux-musleabihf --manifest-path daemon/Cargo.toml

# Emulate execution under QEMU
qemu-arm daemon/target/armv7-unknown-linux-musleabihf/release/kmatrixd



Local Integration & Testing

Test Matrix Server

Run a local matrix-conduit instance on 127.0.0.1:6167 for end-to-end testing:

# Start local conduit test server
./scripts/testserver.sh start

# Provision test accounts
./scripts/testserver.sh register alice hunter2
./scripts/testserver.sh register bob hunter2

# Query status and stop
./scripts/testserver.sh status
./scripts/testserver.sh stop



Headless IPC Testing

Verify IPC non-blocking reads, frame buffering, and correlation without target e-ink hardware:

mkdir -p /tmp/kt/kmatrix
./daemon/target/release/kmatrixd --data-dir /tmp/kt/kmatrix &
luajit scripts/ipctest.lua /tmp/kt



(Requires luasocket and dkjson or standard KOReader JSON modules on LUA_PATH).
