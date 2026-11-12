# 非官方修補版 hbbs 部署說明（香港繁體）

這是**非官方修補版 `hbbs`**，不是 RustDesk 官方程式，也不是 RustDesk Server Pro。
只替換 **hbbs 二進位檔**。現場的 `hbbr` 維持不變，這個壓縮檔**沒有** `hbbr`。

適用環境：Proxmox LXC，`/opt/rustdesk` 內的官方獨立二進位檔（不是 dpkg），`hbbs` 1.1.15，啟動時沒有額外參數（沒有 `-k`），工作目錄是 `/opt/rustdesk`。

## 這個版本做了甚麼

已登入 API 帳號的官方客戶端（約 1.4.1 起，包括 1.5.x）會等伺服器送出已簽名的 `KeyExchange`。上游開源 `hbbs` 從不送出，客戶端約 18 秒後失敗，錯誤是：

`Failed to secure tcp: deadline has elapsed`

本版在主 rendezvous TCP 埠（21116）於簽署金鑰 `sk` 可用時送出 v0 `KeyExchange`。沒有 `-k` 時，`hbbs` 仍會從工作目錄的 `id_ed25519` 載入 `sk`；即使金鑰字串是空的，也會送出握手。未登入或較舊的客戶端行為不變。

`hbbs --version` 應顯示：

```text
hbbs 1.1.15-keyexchange
```

## 來源

- 官方 tag `1.1.15`，commit `9bae9f2f39d92c4b4ba2e28e089da5071897b22e`
- 釘選的 `libs/hbb_common`：`83419b6549636ee39dacef7776c473f5802e08d6`（沒有升級）
- v0 握手移植自尚未合併的 PR：
  - https://github.com/rustdesk/rustdesk-server/pull/745 （Lyapnev，`cbd5687c58cc2a6f991a52ad18bab94d4a53faa9`；只用 v0，沒有採用當中的 WebSocket `OnlineRequest`）
  - https://github.com/rustdesk/rustdesk-server/pull/706 （Silvarion，`60dfed8c3fb80198d99eb085db0a353d28ea4b64`；測試與不要用空的 `-k` 字串關掉握手的理由）
- 授權仍是 GNU AGPL-3.0。這不是官方 RustDesk，也不是 RustDesk Server Pro。

## 不要動這些檔案

替換之前先核對。以下檔案**不要覆寫、不要刪除**：

- `id_ed25519`
- `id_ed25519.pub`
- sqlite 資料庫（`db_v2.sqlite3`，以及旁邊的 `-wal`、`-shm`）
- `hbbr`
- `rustdesk-utils`
- `rustdesk-console`（如果目錄裡有）

只更換 `hbbs` 這一個二進位檔。

## 部署

在 LXC 裡操作。先把新的 `hbbs` 放到主機上（例如 `/root/hbbs-keyexchange/hbbs`），再執行：

```bash
cd /opt/rustdesk
sudo systemctl stop rustdesksignal.service

# 先備份現有二進位檔。檔名保留日期，方便還原。
sudo cp -a hbbs "hbbs.bak-1.1.15.$(date +%Y%m%d%H%M%S)"

# 只覆蓋 hbbs。不要用整個目錄去蓋掉 /opt/rustdesk。
sudo install -m 755 /root/hbbs-keyexchange/hbbs /opt/rustdesk/hbbs

sudo systemctl start rustdesksignal.service
sudo systemctl status rustdesksignal.service --no-pager
```

`install` 只寫入 `hbbs`，不會碰到同一目錄的金鑰、資料庫或 `hbbr`。

## 核對

```bash
cd /opt/rustdesk
./hbbs --version
sha256sum hbbs
```

預期版本字串是 `hbbs 1.1.15-keyexchange`。

本壓縮檔內的 `hbbs` sha256（`sha256sum -c hbbs.sha256`）：

```text
ee9a4bc82f7435ccf6be1e20ec020e08141fda112f0d790ee333099334802cdc  hbbs
```

壓縮檔本身的 sha256 放在壓縮檔**旁邊**的
`rustdesk-hbbs-1.1.15-keyexchange-x86_64-unknown-linux-musl.tar.gz.sha256`。
校驗值不能寫進壓縮檔裡面，否則檔案內容一改，校驗值就會變。

部署完成後，用**已登入**的官方客戶端再連一次。先前的
`Failed to secure tcp: deadline has elapsed` 應不再出現，連線可以建立。
未登入的客戶端應與換檔之前一樣。

伺服器日誌若出現 `key exchange ok from ...`，表示這次 TCP 握手已完成。

## 還原

還原備份的二進位檔即可，不必動金鑰或資料庫：

```bash
cd /opt/rustdesk
sudo systemctl stop rustdesksignal.service
sudo cp -a hbbs.bak-1.1.15.YYYYMMDDHHMMSS hbbs
sudo systemctl start rustdesksignal.service
./hbbs --version
```

還原後版本應回到原來的 `1.1.15`（沒有 `-keyexchange` 字尾）。
