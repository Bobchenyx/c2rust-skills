# C Library to Rust Crate Mapping Reference

Common C libraries and their Rust equivalents or binding crates.

## Legend

| Column | Meaning |
|--------|---------|
| C Library | The C library name (as used with `-l` linker flag or `#include`) |
| Rust Pure | Pure Rust implementation (no C dependency) |
| Rust -sys | Rust bindings wrapping the C library |
| Recommendation | Which to prefer and why |

---

## Standard Library / POSIX

| C Library | Rust Pure | Rust -sys | Recommendation |
|-----------|-----------|-----------|----------------|
| libc (stdio, stdlib) | `std` | `libc` | Use `std` — native Rust equivalents for most functions |
| libm (math) | `std::f64` / `libm` | - | Use `std` math methods; `libm` crate for no_std |
| pthreads | `std::thread`, `rayon` | - | `std::thread` for 1:1 mapping; `rayon` for data parallelism |
| POSIX signals | `signal-hook` | `nix` | `signal-hook` for safe signal handling |
| POSIX sockets | `std::net`, `socket2` | `nix` | `std::net` for TCP/UDP; `socket2` for advanced options |
| POSIX file I/O | `std::fs` | `nix` | Use `std::fs`; `nix` for low-level (ioctl, mmap) |
| dlfcn (dlopen) | `libloading` | `libloading` | `libloading` is the standard choice |
| iconv | `encoding_rs` | `iconv` | `encoding_rs` preferred (pure Rust, fast) |

## Cryptography

| C Library | Rust Pure | Rust -sys | Recommendation |
|-----------|-----------|-----------|----------------|
| OpenSSL | `rustls` + `ring` | `openssl` | `rustls` for new code (memory safe); `openssl` crate for compatibility |
| libsodium | `sodiumoxide`* | `libsodium-sys` | `sodiumoxide` wraps libsodium; for pure Rust consider `orion` |
| GnuTLS | `rustls` | - | Use `rustls` as replacement |
| mbedTLS | `rustls` | `mbedtls` | `rustls` preferred for pure Rust |
| libgcrypt | `ring`, `sha2`, `aes` | - | RustCrypto crates (`sha2`, `aes`, `hmac`, etc.) |

## Compression

| C Library | Rust Pure | Rust -sys | Recommendation |
|-----------|-----------|-----------|----------------|
| zlib | `flate2` (miniz) | `flate2` (zlib backend) | `flate2` with default miniz_oxide backend (pure Rust) |
| libbz2 | `bzip2` | `bzip2` (C backend) | `bzip2` crate supports both backends |
| liblzma / xz | `xz2`, `liblzma` | `lzma-sys` | `xz2` wraps lzma-sys |
| libzstd | `zstd` | `zstd` | `zstd` crate wraps C library (fast) |
| lz4 | `lz4_flex` | `lz4` | `lz4_flex` for pure Rust; `lz4` for C binding |
| snappy | `snap` | - | `snap` is pure Rust |

## Networking

| C Library | Rust Pure | Rust -sys | Recommendation |
|-----------|-----------|-----------|----------------|
| libcurl | `reqwest`, `ureq` | `curl` | `reqwest` (async) or `ureq` (sync blocking) for pure Rust |
| libevent | `tokio`, `mio` | - | `tokio` for async runtime; `mio` for low-level |
| libev | `tokio` | - | Use `tokio` |
| libuv | `tokio` | - | Use `tokio` as async runtime replacement |
| nghttp2 | `h2` | `libnghttp2-sys` | `h2` for pure Rust HTTP/2 |
| libssh2 | `thrussh`, `russh` | `ssh2` | `russh` for pure Rust; `ssh2` wraps libssh2 |
| libpcap | - | `pcap` | `pcap` crate wraps libpcap |
| libmicrohttpd | `hyper`, `axum`, `actix-web` | - | Use any Rust web framework |

## Data Formats

| C Library | Rust Pure | Rust -sys | Recommendation |
|-----------|-----------|-----------|----------------|
| jansson / cJSON | `serde_json` | - | `serde_json` — de facto standard |
| libxml2 | `quick-xml`, `roxmltree` | `libxml` | `quick-xml` for speed; `roxmltree` for DOM |
| expat | `quick-xml` | - | Use `quick-xml` |
| libyaml | `serde_yaml` | `yaml-rust2` | `serde_yaml` for serde integration |
| protobuf-c | `prost`, `protobuf` | - | `prost` (idiomatic) or `protobuf` (Google-backed) |
| msgpack-c | `rmp-serde` | - | `rmp-serde` for serde integration |
| libcsv | `csv` | - | `csv` crate by BurntSushi |
| toml parser | `toml` | - | `toml` crate |

## Database

| C Library | Rust Pure | Rust -sys | Recommendation |
|-----------|-----------|-----------|----------------|
| libsqlite3 | - | `rusqlite` | `rusqlite` wraps sqlite3 C library |
| libpq (PostgreSQL) | `tokio-postgres` | `postgres` | `postgres` (sync) or `tokio-postgres` (async); `sqlx` for compile-time checked queries |
| libmysqlclient | - | `mysql` | `mysql` crate; or `sqlx` with mysql feature |
| hiredis (Redis) | `redis` | - | `redis` crate |
| lmdb | - | `lmdb-rkv`, `heed` | `heed` for safe wrapper |
| leveldb | - | `leveldb`, `rusty-leveldb` | `rusty-leveldb` for pure Rust |

## Image / Multimedia

| C Library | Rust Pure | Rust -sys | Recommendation |
|-----------|-----------|-----------|----------------|
| libpng | `png` | - | `png` crate or `image` for multi-format |
| libjpeg | `jpeg-decoder` | `mozjpeg-sys` | `image` crate handles JPEG |
| giflib | `gif` | - | `gif` crate |
| FFmpeg | - | `ffmpeg-next` | `ffmpeg-next` wraps FFmpeg C libraries |
| SDL2 | - | `sdl2` | `sdl2` crate wraps SDL2 |

## Regex / Text

| C Library | Rust Pure | Rust -sys | Recommendation |
|-----------|-----------|-----------|----------------|
| PCRE / PCRE2 | `regex` | `pcre2` | `regex` for pure Rust (fast, safe); `pcre2` for PCRE compatibility |
| ICU | - | `icu`, `rust_icu` | `icu` crate for Unicode |
| readline | `rustyline` | - | `rustyline` is pure Rust readline |
| ncurses | `crossterm`, `termion` | `ncurses` | `crossterm` for cross-platform terminal |

## System / OS

| C Library | Rust Pure | Rust -sys | Recommendation |
|-----------|-----------|-----------|----------------|
| libsystemd | - | `systemd` | `systemd` crate |
| libdbus | `zbus` | `dbus` | `zbus` for pure Rust D-Bus |
| libudev | - | `udev` | `udev` crate |
| libnotify | `notify-rust` | - | `notify-rust` |
| inotify | `notify`, `inotify` | - | `notify` for cross-platform file watching |

## Logging

| C Library | Rust Pure | Rust -sys | Recommendation |
|-----------|-----------|-----------|----------------|
| syslog | `syslog` | - | `syslog` crate, or use `tracing` + `tracing-journald` |
| log4c | `log` + `env_logger` | - | `log` facade + any backend (`env_logger`, `tracing`) |
| No logging | `tracing` | - | `tracing` is the modern standard for instrumentation |

## Testing

| C Framework | Rust Equivalent | Notes |
|------------|----------------|-------|
| CUnit | `#[test]` + `assert!` | Built-in to Rust |
| Check | `#[test]` | Built-in |
| cmocka | `mockall` | For mocking |
| Unity | `#[test]` | Built-in |
| Google Test | `#[test]` + `rstest` | `rstest` for parameterized tests |

---

## Decision Guide

When choosing between pure Rust and -sys bindings:

**Prefer pure Rust when**:
- Security is important (memory safety guarantees)
- Cross-compilation is needed (no C toolchain dependency)
- The pure Rust version is mature and well-maintained
- Performance is comparable

**Prefer -sys bindings when**:
- The C library has no mature Rust equivalent
- Exact behavior compatibility with C version is required
- The C library is heavily optimized (e.g., OpenSSL for FIPS compliance)
- Gradual migration — keep using the same C library through Rust bindings first

**Incremental conversion tip**: Start with -sys bindings to maintain behavioral compatibility, then migrate to pure Rust equivalents module by module.
