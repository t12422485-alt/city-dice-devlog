# 04 VPS 部署教學

目標：**電腦關機後，朋友打開固定的 HTTPS 網址就能玩**。

本專案在 2026-10-21～22 依照下面的步驟，部署到一台 **Debian 13、1 vCPU、512 MB 記憶體、沒有 Docker** 的 VPS，並完成驗證（結果見最後一節）。

## 範例值

以下都用範例，請換成自己的：

| 項目 | 範例 |
|---|---|
| 主機 IP | `203.0.113.10` |
| 網址 | `your-game.duckdns.org` |
| 打包檔 | `city-dice-<提交>.tar.gz`（`<提交>` 是 Git 提交編號，例如 `abc1234`） |
| 登入帳號 | `root`（一般帳號請在 VPS 指令前加 `sudo`） |

## 標示說明

每段指令都標明在哪裡執行：

- 🪟 **Windows PowerShell**：在自己的電腦執行。
- 🐧 **VPS SSH**：用 SSH 工具（例如 Windows 內建的 `ssh`、WindTerm、PuTTY）登入 VPS 後執行。
- 🌐 **瀏覽器**：在網站上操作。

---

## 架構

```
朋友的瀏覽器 ──HTTPS／WSS（443）──▶ Caddy ──▶ 127.0.0.1:3000 遊戲伺服器（systemd：city-dice）
                                    │                          │
                     自動申請／更新 Let's Encrypt 憑證        房間存檔 /var/lib/city-dice
```

- 遊戲伺服器**只聽 VPS 內部**的 `127.0.0.1:3000`，外面只能透過 Caddy 連進來。
- 以不能登入的系統帳號 `citydice` 執行；開機自動啟動，異常結束 3 秒後自動重啟。
- 房間存檔放在 `/var/lib/city-dice`：重啟後恢復，對局先暫停，玩家重連後繼續。

---

## 步驟 0：只讀檢查（不修改任何東西）

🐧 **VPS SSH**

```bash
cat /etc/os-release | head -3; nproc; free -m; df -h /
ufw status; iptables -S INPUT | head -5
ss -lntup
systemctl list-units --type=service --state=running --no-pager --plain
```

要特別看兩件事：

1. **80、443 有沒有被其他程式占用**：`ss` 的輸出裡不應該有它們。
2. **有沒有要保留的既有設定**：例如 SSH、VPN（WireGuard）。如果 INPUT 裡有不屬於 ufw 的規則，先查出它從哪裡來，確認重開機後會自動加回：

```bash
grep -rIl 51820 /etc 2>/dev/null                                           # 51820 換成你要查的連接埠
grep -rIn 51820 /etc 2>/dev/null | grep -viE 'privatekey|presharedkey'     # 過濾掉金鑰行，不要把金鑰貼給任何人
systemctl list-unit-files --no-pager | grep -iE 'iptables|nftables|wg|firewall|ufw'
```

> 本專案的 VPS 就有一條 VPN 規則是由自訂的 systemd 服務在開機時加入的。確認了它的啟動順序在 ufw 之後，才放心進行後面的重開機測試。

---

## 步驟 1：固定網址（DuckDNS，免費）

🌐 **瀏覽器**

1. 到 <https://www.duckdns.org>，用 GitHub 或 Google 帳號登入。
2. 在 **sub domain** 輸入名稱（例如 `your-game`），按 **add domain**。
3. 在 **current ip** 填 VPS 的 IP，按 **update ip**。
4. DuckDNS 頁面上的 **token 不要貼給別人**，也不要放進專案。

🪟 **Windows PowerShell**：確認網址指向你的 VPS

```powershell
nslookup your-game.duckdns.org 1.1.1.1
nslookup your-game.duckdns.org 8.8.8.8
```

兩個都應該顯示你的 VPS IP。如果你有自己的網域，也可以加一筆 A 紀錄指向 VPS IP。

---

## 步驟 2：安裝 Node.js（官方版本，核對 SHA256）

🐧 **VPS SSH**

```bash
cd /tmp
curl -fsSLO https://nodejs.org/dist/latest-v24.x/SHASUMS256.txt
F=$(grep -oE 'node-v24\.[0-9]+\.[0-9]+-linux-x64\.tar\.xz' SHASUMS256.txt | head -1); echo "檔案：$F"
curl -fsSLO "https://nodejs.org/dist/latest-v24.x/$F"
grep " $F\$" SHASUMS256.txt | sha256sum -c - && tar -xJf "$F" -C /opt && ln -sfn "/opt/${F%.tar.xz}" /opt/node && /opt/node/bin/node --version
```

- 必須看到 `...tar.xz: OK`，最後一行是 `v24.x.x`。
- 用 `&&` 串起來：**核對失敗就不會解壓**。
- 不用發行版內建的舊版 Node.js（Debian 13 內建的 v20 已停止支援）。

---

## 步驟 3：建立服務帳號與資料夾

🐧 **VPS SSH**

```bash
id citydice 2>/dev/null || useradd --system --home-dir /nonexistent --no-create-home --shell /usr/sbin/nologin citydice
mkdir -p /opt/city-dice/releases
id citydice; ls -ld /opt/city-dice/releases
```

`citydice` 是**不能登入**的系統帳號，只用來執行遊戲。就算遊戲程式出問題，也碰不到其他東西。

---

## 步驟 4：在 Windows 打包，上傳到 VPS

### 為什麼在 Windows 打包

這個遊戲的相依套件都是純 JavaScript，在 Windows 安裝好一起打包，VPS 就不需要 npm，也不用編譯。

### 打包和上傳

🪟 **Windows PowerShell**（在專案資料夾）

```powershell
powershell -ExecutionPolicy Bypass -File .\deploy\vps\pack.ps1
Copy-Item deploy\vps\city-dice.service, deploy\vps\Caddyfile dist\
Get-FileHash dist\* -Algorithm SHA256
scp dist\city-dice-<提交>.tar.gz dist\city-dice.service dist\Caddyfile root@203.0.113.10:/tmp/
```

- `pack.ps1` 只打包**已提交**的版本，加上正式環境需要的 `node_modules`，不包含測試、截圖和存檔。
- 執行 `scp` 時，VPS 的**密碼或 SSH 金鑰請自己輸入**，不要貼到任何對話裡。
- `scp` 不能用的話，可以用 SSH 工具內建的 SFTP，把檔案拖進 VPS 的 `/tmp/`。

### 核對上傳的檔案

🐧 **VPS SSH**：把 `<…>` 換成 Windows 上 `Get-FileHash` 顯示的值，注意中間是**兩個空格**。

```bash
cd /tmp && sha256sum -c <<'EOF'
<打包檔的 SHA256>  city-dice-<提交>.tar.gz
<service 檔的 SHA256>  city-dice.service
<Caddyfile 的 SHA256>  Caddyfile
EOF
```

3 行都必須是 `OK`。

> ⚠️ Debian 13 的 `/tmp` 放在記憶體裡，**重開機會清空**。請在同一次開機內完成下一步；如果中間重開過，要重新上傳。

---

## 步驟 5：安裝遊戲、設定存檔、啟動常駐服務

🐧 **VPS SSH**

```bash
REL=city-dice-<提交>
tar -xzf /tmp/$REL.tar.gz -C /opt/city-dice/releases && chmod -R a+rX /opt/city-dice/releases/$REL \
 && ln -sfn /opt/city-dice/releases/$REL /opt/city-dice/current \
 && install -m 644 /tmp/city-dice.service /etc/systemd/system/city-dice.service \
 && systemctl daemon-reload && systemctl enable --now city-dice && sleep 4
systemctl --no-pager status city-dice | head -12
echo "開機自動啟動：$(systemctl is-enabled city-dice)"
echo "健康檢查：$(curl -s http://127.0.0.1:3000/healthz)"
ls -ld /var/lib/city-dice
journalctl -u city-dice --no-pager -n 5
```

應該看到：

- `active (running)`
- `開機自動啟動：enabled`
- `健康檢查：ok`
- 日誌有 `房間存檔：/var/lib/city-dice`

### `city-dice.service` 的重點設定

```ini
[Service]
User=citydice
WorkingDirectory=/opt/city-dice/current
Environment=NODE_ENV=production
Environment=HOST=127.0.0.1
Environment=PORT=3000
Environment=DATA_DIR=/var/lib/city-dice
Environment=NODE_OPTIONS=--max-old-space-size=192
ExecStart=/opt/node/bin/node server/index.js
Restart=always
RestartSec=3
MemoryHigh=240M
MemoryMax=300M
StateDirectory=city-dice
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true

[Install]
WantedBy=multi-user.target
```

| 設定 | 作用 |
|---|---|
| `Restart=always`、`RestartSec=3` | 異常結束 3 秒後自動重啟 |
| `WantedBy=multi-user.target`（搭配 `enable`） | 開機自動啟動 |
| `StateDirectory=city-dice` | systemd 自動建立 `/var/lib/city-dice`，並交給 `citydice` 帳號使用 |
| `MemoryHigh`、`MemoryMax`、`--max-old-space-size` | 小記憶體主機上限制用量 |
| `ProtectSystem=strict` 等 | 程式只能寫自己的存檔資料夾 |

---

## 步驟 6：開放防火牆、安裝 Caddy、設定 HTTPS

### 6-1 開放 80、443

🐧 **VPS SSH**（以 ufw 為例）

```bash
ufw status
ufw allow 80/tcp && ufw allow 443/tcp      # 只新增，SSH（22）和其他既有規則不動
ufw status
```

**先確認 `22/tcp` 一直是 ALLOW，才不會把自己鎖在外面。**

如果主機商的網頁後台也有防火牆（例如 Vultr 的 Firewall 群組），那裡也要開放 TCP 80、443。

### 6-2 安裝 Caddy、申請憑證

🐧 **VPS SSH**

```bash
apt-get update && apt-get install -y caddy
sed -e 's/GAME_HOST/your-game.duckdns.org/' -e 's/EMAIL_FOR_CERT_NOTICES/you@example.com/' /tmp/Caddyfile > /etc/caddy/Caddyfile
caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile && systemctl reload caddy
sleep 25
journalctl -u caddy --no-pager -n 80 | grep -iE 'certificate obtained|error|challenge|failed' | tail -8
curl -sI http://your-game.duckdns.org | head -3          # 應該是 308 轉到 https
curl -s https://your-game.duckdns.org/healthz; echo     # ok
```

- 不想收憑證到期通知，可以把 `email` 那一行刪掉：用 `-e '/email EMAIL_FOR_CERT_NOTICES/d'` 取代第二個 `-e`。
- Caddy 會**自動申請並更新** Let's Encrypt 憑證，HTTP 自動轉 HTTPS；WebSocket 不需要額外設定。
- 日誌出現 `no OCSP server specified in certificate` 可以忽略：Let's Encrypt 已停止提供 OCSP，不影響 HTTPS。
- 憑證申請失敗，最常見的原因是 80 或 443 被防火牆擋住（主機上的防火牆或主機商後台的防火牆）。

### Caddyfile 內容（節錄，完整版另有日誌設定）

```caddyfile
{
	email EMAIL_FOR_CERT_NOTICES
}

GAME_HOST {
	reverse_proxy 127.0.0.1:3000 {
		lb_try_duration 10s
	}
	header {
		Strict-Transport-Security "max-age=31536000"
		-Server
	}
}
```

---

## 步驟 7：驗證（全部通過才算完成）

### 7-1 從外部網路測試

🪟 **Windows PowerShell**：在自己的電腦（不是 VPS 上）測試。

```powershell
curl.exe -sI http://your-game.duckdns.org/            # 308 → https
curl.exe -s https://your-game.duckdns.org/healthz     # ok
```

接著用瀏覽器打開 `https://your-game.duckdns.org/`：建立房間，用另一台裝置（例如手機網路）輸入房間代碼加入，開局後擲骰，確認兩邊看到同一個結果。

### 7-2 重啟、當機、重開機

🐧 **VPS SSH**

```bash
# 正常重啟（更新版本時就是這樣）
systemctl restart city-dice; sleep 5; systemctl is-active city-dice

# 模擬當機：強制結束程式，應該 3 秒後自動重啟，NRestarts 加 1
systemctl show city-dice -p MainPID
systemctl kill -s KILL city-dice; sleep 7
systemctl show city-dice -p MainPID -p NRestarts

# 重開機（SSH 會斷線，1～2 分鐘後重新登入）
systemctl reboot
```

重開機後重新登入，檢查：

```bash
systemctl --failed --no-pager --plain                   # 應該沒有項目
for s in city-dice caddy ufw ssh; do echo "$s：$(systemctl is-active $s)"; done
curl -s https://your-game.duckdns.org/healthz; echo
journalctl -u city-dice -b --no-pager -n 6              # 「恢復了 N 間房間」
```

有其他既有服務（例如 VPN）的話，也要實際連線確認。

### 本專案的實際驗證結果（2026-10-21～22）

| 驗證 | 方法 | 結果 |
|---|---|---|
| 打包檔在乾淨資料夾、用正式環境變數啟動 | Windows 本機 | 13/13 通過 |
| 外部網路：憑證、HTTP 轉 HTTPS、首頁與 3D 資源、伺服器程式與 `.env` 等不公開（404）、**8 個連線**建房加入開局、第 9 人被拒絕、擲骰同步 | Windows → 網際網路 → VPS | 25/25 通過（部署後，以及重開機後各一次） |
| 真實瀏覽器打開正式網址：3D 畫面、建房、開局、沒有頁面錯誤 | 無頭 Chrome | 通過 |
| 兩位玩家對局中：服務重啟（中斷約 0.9 秒）、強制結束（約 3.9 秒）、VPS 重開機（約 46 秒）；每次都核對用原本的憑證回房、先暫停、到齊後繼續、版本不倒退、現金與位置完全相同、可繼續擲骰、重送被拒絕 | Windows 上的監看程式 | 28/28 通過 |
| 重開機後：沒有啟動失敗的服務、HTTPS 正常、房間自動恢復、VPN 規則與實際連線正常 | VPS＋作者的裝置 | 通過 |

> 「8 個連線」是程式模擬的；8 位真人用不同裝置長時間同時遊玩尚未實測。

---

## 之後的維護

### 更新版本

1. 🪟 在 Windows 提交新版本 → `pack.ps1` → 上傳 → 核對 SHA256。
2. 🐧 在 VPS 解壓到新的 `releases/<版本>`，`ln -sfn` 讓 `current` 指向它，`systemctl restart city-dice`。

房間會恢復，玩家畫面會自動重連。

### 退回舊版

把 `current` 指回舊版本，再 `systemctl restart city-dice`。

### 備份與還原存檔

🐧 **VPS SSH**

```bash
tar -czf city-dice-data-$(date +%F).tar.gz -C /var/lib city-dice            # 備份（執行中也可以）
systemctl stop city-dice && tar -xzf city-dice-data-YYYY-MM-DD.tar.gz -C /var/lib \
 && chown -R citydice:citydice /var/lib/city-dice && systemctl start city-dice   # 還原
```

存檔裡有玩家的重連憑證，不要公開分享。

### 小記憶體主機的建議（選擇性）

本專案的 VPS 重開機後可用記憶體約 295 MiB，沒有交換檔也能跑。如果想多一層保險，可以建立 1 GB 交換檔：

```bash
fallocate -l 1G /swapfile && chmod 600 /swapfile && mkswap /swapfile && swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

本專案目前**沒有**建立交換檔。
