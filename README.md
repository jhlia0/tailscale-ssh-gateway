# Tailscale SSH Gateway

使用 Docker 架設 Tailscale，讓遠端主機能夠透過 Tailscale 網路連線到本地機器的 SSH。

## 功能特點

- 使用 Docker 容器執行 Tailscale
- 自動啟用 Tailscale SSH 功能
- 持久化 Tailscale 狀態
- 使用 host 網路模式，直接訪問宿主機服務

## 前置要求

- Docker 和 Docker Compose 已安裝
- 本地機器已啟動 SSH 服務
- Tailscale 賬號（免費賬號即可）

## 快速開始

### 1. 獲取 Tailscale Auth Key

1. 訪問 [Tailscale Admin Console](https://login.tailscale.com/admin/settings/keys)
2. 點選 "Generate auth key"
3. 建議選項：
   - ✅ Reusable（可重複使用）
   - ✅ Tags: ssh-gateway（選擇此標籤，用於 ACL 控制）
   - ⚠️  **不要選擇 Ephemeral**（使用標籤時不能是臨時金鑰）
4. 複製生成的 auth key

**注意**：如果你不需要使用 ACL 限制出站訪問，可以移除 `docker-compose.yml` 中的 `--advertise-tags=tag:ssh-gateway` 參數，這樣就可以使用 Ephemeral key。

### 2. 配置環境變數

```bash
# 複製環境變數示例檔案
cp .env.example .env

# 編輯 .env 檔案，填入你的 auth key
# TS_AUTHKEY=tskey-auth-xxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### 3. 啟動容器

```bash
# 啟動 Tailscale 容器
docker-compose up -d

# 檢視日誌
docker-compose logs -f
```

### 4. 驗證連線

在啟動容器後，你應該能在 [Tailscale Admin Console](https://login.tailscale.com/admin/machines) 看到新裝置 `ssh-gateway`。

從另一臺已連線到同一 Tailscale 網路的裝置上測試 SSH 連線：

```bash
# 獲取 Tailscale 分配的 IP 地址
# 在 Tailscale Admin Console 中檢視，或在容器中執行：
docker exec tailscale-ssh-gateway tailscale ip -4

# 從遠端主機連線
ssh your-username@100.x.x.x
```

## 使用 Tailscale SSH

Tailscale 內建了 SSH 功能，配置中已啟用 `--ssh` 引數。這樣可以：

1. **無需暴露 22 埠**：只有 Tailscale 網路內的裝置可以訪問
2. **使用 Tailscale 身份驗證**：可以使用 Tailscale 的 ACL 控制訪問許可權
3. **自動金鑰管理**：無需手動管理 SSH 金鑰

### 透過 Tailscale SSH 連線

```bash
# 使用裝置名連線（推薦）
ssh ssh-gateway

# 或使用 Tailscale IP
ssh 100.x.x.x
```

## 配置說明

### docker-compose.yml 關鍵配置

```yaml
environment:
  - TS_AUTHKEY=${TS_AUTHKEY}     # Tailscale 認證金鑰
  - TS_STATE_DIR=/var/lib/tailscale  # 狀態儲存目錄
  - TS_USERSPACE=false            # 使用核心網路棧
  - TS_ACCEPT_DNS=false           # 不接受 Tailscale DNS
  - TS_EXTRA_ARGS=--ssh --advertise-tags=tag:ssh-gateway  # 啟用 SSH 並設定標籤

volumes:
  - ./tailscale-data:/var/lib/tailscale  # 持久化儲存
  - /dev/net/tun:/dev/net/tun     # TUN 裝置

cap_add:
  - NET_ADMIN                     # 網路管理許可權
  - SYS_MODULE                    # 核心模組載入許可權

network_mode: host                # 使用主機網路
```

## 常用命令

```bash
# 檢視容器狀態
docker-compose ps

# 檢視日誌
docker-compose logs -f tailscale

# 檢視 Tailscale 狀態
docker exec tailscale-ssh-gateway tailscale status

# 檢視 Tailscale IP
docker exec tailscale-ssh-gateway tailscale ip

# 重啟容器
docker-compose restart

# 停止容器
docker-compose down

# 完全清理（包括資料）
docker-compose down -v
rm -rf tailscale-data
```

## 故障排除

### 1. 容器無法啟動

檢查是否有 `/dev/net/tun` 裝置：
```bash
ls -l /dev/net/tun
```

如果不存在，建立它：
```bash
sudo mkdir -p /dev/net
sudo mknod /dev/net/tun c 10 200
sudo chmod 666 /dev/net/tun
```

### 2. 無法連線 SSH

確認本地 SSH 服務正在執行：
```bash
# macOS
sudo systemsetup -getremotelogin
sudo systemsetup -setremotelogin on

# Linux
sudo systemctl status sshd
sudo systemctl start sshd
```

### 3. Tailscale 認證失敗

- 檢查 `.env` 檔案中的 `TS_AUTHKEY` 是否正確
- 確認 auth key 未過期
- 如果使用 ephemeral key，確保之前的容器已被刪除

## 安全建議

1. **限制出站訪問**：使用標籤和 ACL 配置，確保 SSH gateway 只能被訪問，不能主動訪問其他裝置（參見「進階配置」章節）
2. **配置嚴格的 ACL**：在 Tailscale Admin Console 中配置訪問控制列表，只允許特定使用者訪問
3. **定期更新**：保持 Tailscale 容器映象更新
4. **最小許可權原則**：只授予必要的 SSH 訪問許可權
5. **監控日誌**：定期檢查 SSH 和 Tailscale 日誌
6. **定期審核裝置**：在 Tailscale Admin Console 中定期檢查連線的裝置

## 進階配置

### 限制 SSH Gateway 的出站訪問（只允許被訪問）

為了安全性，你可能希望這個 SSH gateway 只能被其他裝置訪問，但不能主動連線到其他 Tailscale 裝置。這可以透過 Tailscale ACL 配置實現。

配置已經在 `docker-compose.yml` 中添加了標籤 `tag:ssh-gateway`，現在需要在 [Tailscale ACL 頁面](https://login.tailscale.com/admin/acls) 配置相應的規則：

```json
{
  "tagOwners": {
    "tag:ssh-gateway": ["autogroup:admin"]
  },
  "acls": [
    {
      "action": "accept",
      "src": ["autogroup:member"],
      "dst": ["tag:ssh-gateway:22"]
    },
    {
      "action": "accept",
      "src": ["autogroup:member"],
      "dst": ["tag:ssh-gateway:*"]
    }
  ],
  "ssh": [
    {
      "action": "accept",
      "src": ["autogroup:member"],
      "dst": ["tag:ssh-gateway"],
      "users": ["autogroup:nonroot", "root"]
    }
  ]
}
```

**重要說明**：
- 上述 ACL 配置**只定義了其他裝置訪問 `tag:ssh-gateway` 的規則**
- **沒有定義 `tag:ssh-gateway` 作為源（src）的規則**，因此它無法主動訪問其他裝置
- `autogroup:member` 代表所有 Tailscale 網路中的成員
- 如果你只想讓特定使用者訪問，可以將 `autogroup:member` 改為具體的郵箱地址

**測試驗證**：

從 SSH gateway 容器內部嘗試連線其他裝置應該會失敗：
```bash
# 進入容器
docker exec -it tailscale-ssh-gateway sh

# 嘗試 ping 其他 Tailscale 裝置（應該會被 ACL 阻止）
ping 100.x.x.x

# 嘗試 SSH 到其他裝置（應該會被 ACL 阻止）
ssh 100.x.x.x
```

從其他 Tailscale 裝置訪問這個 gateway 應該會成功：
```bash
# 從其他裝置執行
ssh ssh-gateway
```

### 使用 Tailscale ACL 控制特定使用者訪問

如果你想更精細地控制哪些使用者可以訪問，可以使用以下配置：

```json
{
  "tagOwners": {
    "tag:ssh-gateway": ["autogroup:admin"]
  },
  "acls": [
    {
      "action": "accept",
      "src": ["user@example.com", "another-user@example.com"],
      "dst": ["tag:ssh-gateway:22"]
    }
  ],
  "ssh": [
    {
      "action": "accept",
      "src": ["user@example.com"],
      "dst": ["tag:ssh-gateway"],
      "users": ["your-username"]
    }
  ]
}
```

### 自定義主機名

修改 `docker-compose.yml` 中的 `hostname` 引數：

```yaml
hostname: my-custom-name
```

## 參考資料

- [Tailscale 官方文件](https://tailscale.com/kb/)
- [Tailscale SSH 文件](https://tailscale.com/kb/1193/tailscale-ssh/)
- [Docker 官方文件](https://docs.docker.com/)

## 許可證

MIT
