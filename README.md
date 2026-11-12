# Unofficial hbbs 1.1.15 with server-side `secure_tcp` (v0)

This is an **unofficial patched `hbbs`**, not official RustDesk and not RustDesk Server Pro.
It is a private port of the open-source rendezvous server so that official RustDesk **1.4.1+ and 1.5.x** clients which are logged in to an API account can finish the TCP handshake.

`hbbr` is not modified and is not included in the deploy archive. Do not replace a live `hbbr`.

License remains **GNU AGPL-3.0-only**. Copyright remains with the upstream authors. See `LICENSE`.

## What was wrong

Official open-source `hbbs` never sends a signed `KeyExchange`. From official clients around 1.4.1, a logged-in controller (it has an API access token) calls `secure_tcp()` and waits about 18 seconds for that message, then fails with:

`Failed to secure tcp: deadline has elapsed`

Logged-out clients skip the handshake and still work. This is not a rustdesk-console or API bug.

## What this patch does

On the main rendezvous TCP port (21116), when a signing key (`sk`) is loaded, `hbbs` sends a **v0** signed ephemeral public key immediately after accept.

- If the client answers with a sealed session key, the server derives the key with `hbb_common::tcp::Encrypt` and encrypts later traffic on that connection, including replies sent through a stashed TCP sink (`PunchHoleResponse`, `RelayResponse`).
- If the client does not answer with a sealed key (older client, or logged out), that first message is passed to the normal handler unchanged.
- WebSocket connections are left alone.
- No key-exchange v1. `libs/hbb_common` is not bumped.

`hbbs` is often started with no `-k`. `get_server_sk("")` still loads `id_ed25519` but returns an empty key string. This build sends `KeyExchange` when `sk` is present. It does **not** require the key string to be non-empty.

`hbbs --version` reports `1.1.15-keyexchange`.

## Upstream

| Item | Value |
| --- | --- |
| Repository | https://github.com/rustdesk/rustdesk-server |
| Tag | `1.1.15` |
| Commit | `9bae9f2f39d92c4b4ba2e28e089da5071897b22e` |
| `libs/hbb_common` | `83419b6549636ee39dacef7776c473f5802e08d6` (vendored at that pin; not updated) |

The v0 handshake behaviour is ported from unmerged pull requests, not cherry-picked as git commits (those branches sit on later `master` and do not apply cleanly onto 1.1.15):

- https://github.com/rustdesk/rustdesk-server/pull/745 (Lyapnev, head `cbd5687c58cc2a6f991a52ad18bab94d4a53faa9`). v0 `SecureCodec` offer, sealed-key install, and pass-through of a non-exchange first message. The WebSocket `OnlineRequest` hunk from that PR is **not** included.
- https://github.com/rustdesk/rustdesk-server/pull/706 (Silvarion, head `60dfed8c3fb80198d99eb085db0a353d28ea4b64`). Unit-test rationale: signature round-trip, and an encrypted reply through the stashed sink. Also the reason not to gate the offer on an empty `-k` string.

Nothing from `master` after 1.1.15 is included (message-size limits, relay UUID checks, hbbr IPv4 list matching, UDP punch-hole removal, and so on).

The published 1.1.15 Linux amd64 zip matches the live host (`hbbs` sha256 `2f6a808ce7cd523d46d05cb697c974f17c4c1e6d41d4132ed921ed6142c873f4`). That tag's `Cargo.toml` still says `1.1.14`, while the official binary's version string is `1.1.15` (debian changelog is `1.1.15`). This tree sets the package version to `1.1.15-keyexchange` so `hbbs --version` stays on the 1.1.15 line and is distinguishable. `hbbr.rs` is unchanged; this archive does not ship `hbbr`.

## Build locally

Official Linux amd64 workflow (tag 1.1.15): Rust **1.88**, `--release --all-features`, target **`x86_64-unknown-linux-musl`** (`cross`). The release profile is LTO, `codegen-units = 1`, `panic = abort`, `strip = true`. Rust 1.88's musl target links a static PIE.

```bash
rustup toolchain install 1.88.0 --profile minimal --target x86_64-unknown-linux-musl
# C dependencies (libsodium, sqlite) must be musl objects. Example:
export CC_x86_64_unknown_linux_musl=musl-gcc
export SQLITE3_LIB_DIR=/path/to/musl-libsqlite
export SQLITE3_STATIC=1
cargo +1.88.0 test --lib -- --skip test_insert
cargo +1.88.0 build --release --all-features --target x86_64-unknown-linux-musl --bin hbbs
./target/x86_64-unknown-linux-musl/release/hbbs --version
```

Tests cover the v0 round-trip, the empty-string `get_server_sk("")` case, a plain first message, the stashed-sink encrypted `PunchHoleResponse`, and the no-signing-key path.

Deploy notes for the Proxmox host are in `DEPLOY-zh-HK.md` (Traditional Chinese, Hong Kong).

## Shipped build

Rust `1.88.0`, target `x86_64-unknown-linux-musl`, `--release --all-features --bin hbbs`. The binary is a stripped static PIE (`file`: `static-pie linked, stripped`), same description as the official 1.1.15 amd64 `hbbs`. `hbbr` was not built.

| File | sha256 |
| --- | --- |
| `hbbs` | `ee9a4bc82f7435ccf6be1e20ec020e08141fda112f0d790ee333099334802cdc` |
| `rustdesk-hbbs-1.1.15-keyexchange-x86_64-unknown-linux-musl.tar.gz` | `1664893a47a8dc814397c139729dfbb98ceb4d70cd7d2979d498d186d7d5bf38` |

`hbbs --version` prints `hbbs 1.1.15-keyexchange`.

## 說明（繁體中文）

這是**非官方修補版 `hbbs`**，不是 RustDesk 官方程式，也不是 RustDesk Server Pro。只修 rendezvous 伺服器，**不修改、不重編、不分發 `hbbr`**。授權仍是 AGPL-3.0。

已登入 API 帳號的官方客戶端（約 1.4.1 起，包括 1.5.x）會先做 `secure_tcp()`，等伺服器送出已簽名的 `KeyExchange`。上游開源 `hbbs` 從不送這則訊息，客戶端約 18 秒後失敗：`Failed to secure tcp: deadline has elapsed`。未登入的客戶端會跳過這步，所以仍然可用。

本補丁在主 TCP 埠（21116）於 `sk` 可用時立刻送出 **v0** 簽名暫時公鑰。客戶端若回傳密封金鑰，之後的連線（包括稍後經暫存 sink 送出的回應）會用 `hbb_common::tcp::Encrypt` 加密。客戶端若不做交換，第一則訊息照原邏輯處理。WebSocket 不送 `KeyExchange`。沒有實作 v1，也沒有升級 `hbb_common`。

常見的「沒有 `-k`」啟動方式仍會從 `id_ed25519` 載入 `sk`，但 `get_server_sk("")` 的金鑰字串可能是空的。本版以 **有沒有 `sk`** 決定是否送出，不以金鑰字串是否為空作條件。

`hbbs --version` 顯示 `1.1.15-keyexchange`。

來源是官方 tag `1.1.15`（`9bae9f2f39d92c4b4ba2e28e089da5071897b22e`）及釘選的 `hbb_common`（`83419b6549636ee39dacef7776c473f5802e08d6`）。握手行為取自尚未合併的 [PR 745](https://github.com/rustdesk/rustdesk-server/pull/745)（Lyapnev，`cbd5687c58cc2a6f991a52ad18bab94d4a53faa9`，只用 v0，不含 WebSocket `OnlineRequest`）以及 [PR 706](https://github.com/rustdesk/rustdesk-server/pull/706)（Silvarion，`60dfed8c3fb80198d99eb085db0a353d28ea4b64`）的測試與「不要用空的 `-k` 字串關掉握手」的理由。沒有帶入 1.1.15 之後的 master 改動。

部署步驟見 `DEPLOY-zh-HK.md`。
