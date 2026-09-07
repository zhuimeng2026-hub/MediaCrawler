# LAN Deployment Guide — MediaCrawler @ 192.168.20.173

把 MediaCrawler + Chrome 以 systemd 服务的形式跑在 192.168.20.173 上，
让团队里 .20 段其他机器通过 HTTP/WS API 或 CDP 调用。

---

## 1. 架构

```
┌──────────────────── 192.168.20.173 (本机) ─────────────────────┐
│                                                                 │
│   xvfb.service     Xvfb :99 (虚拟显示器 1920×1080×24)            │
│         │                                                         │
│         ▼  DISPLAY=:99                                           │
│   chrome-remote.service  Chrome for Testing 149 (绑 [::1]:9222) │
│         │                                                         │
│         ▼  [::1]:9222                                            │
│   chrome-cdp-forward.service  socat: 0.0.0.0:9222 → [::1]:9222 │
│                                                                 │
│   mediacrawler-api.service  uvicorn FastAPI: 0.0.0.0:8091        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
              │                                                │
              ▼ LAN (.20 subnet，外部硬件防火墙放行)
   其他机器 (e.g. 192.168.20.168)
       ├─ HTTP  → http://192.168.20.173:8091/docs        (Swagger)
       ├─ HTTP  → http://192.168.20.173:8091/api/crawler/start
       └─ WS    → ws://192.168.20.173:9222/devtools/browser/<uuid>
                   (raw CDP，绕过 MediaCrawler 直接控制 Chrome)
```

---

## 2. 前置依赖

| 工具 | 版本 | 用途 |
|---|---|---|
| uv | 0.11.8 | Python 包管理 |
| Python | 3.11 | 运行时 |
| socat | 任意 | TCP 转发器（Chrome 149 不支持绑 0.0.0.0:9222） |
| Xvfb | 任意 | 虚拟显示，让非 headless Chrome 能跑在无图形登录的机器上 |
| Chrome for Testing | 149.0.7827.55 | `/root/.cache/ms-playwright/chromium-1228/chrome-linux64/chrome`（playwright 安装） |

软件防火墙**已停**（外部硬件防火墙接管）：

```bash
iptables -S INPUT            # 只有 -P INPUT ACCEPT
systemctl status netfilter-persistent  # inactive
# 备份：/etc/iptables/rules.v4.bak-<timestamp>
```

---

## 3. 文件清单

| 路径 | 用途 |
|---|---|
| `/opt/MediaCrawler/config/base_config.py` | `ENABLE_CDP_MODE=True`，`CDP_HOST="localhost"`，`CDP_DEBUG_PORT=9222` |
| `/opt/MediaCrawler/api/main.py` | CORSMiddleware 加 `allow_origin_regex=r"http://192\.168\.20\.\d+(:\d+)?"` |
| `/etc/systemd/system/xvfb.service` | Xvfb :99 |
| `/etc/systemd/system/chrome-remote.service` | Chrome for Testing 149 |
| `/etc/systemd/system/chrome-cdp-forward.service` | socat TCP 转发 |
| `/etc/systemd/system/mediacrawler-api.service` | uvicorn :8091 |
| `/opt/chrome-remote/` | Chrome 用户数据目录（持久登录态） |

---

## 4. 一键部署

```bash
# 0. 安装 MediaCrawler（前置，详见 README.md）
cd /opt && git clone git@github.com:zhuimeng2026-hub/MediaCrawler.git
cd MediaCrawler && uv sync && uv run playwright install chromium

# 1. Chrome 数据目录
mkdir -p /opt/chrome-remote

# 2. 写 4 个 unit 文件（内容见 §5）

# 3. 启用 + 启动
systemctl daemon-reload
systemctl enable --now \
  xvfb.service \
  chrome-remote.service \
  chrome-cdp-forward.service \
  mediacrawler-api.service

# 4. 验证
systemctl status xvfb.service chrome-remote.service \
  chrome-cdp-forward.service mediacrawler-api.service
ss -tlnp | grep -E ':(8091|9222)'

curl -s http://192.168.20.173:9222/json/version | head -c 200
curl -s http://192.168.20.173:8091/api/health
```

期望输出：

```
LISTEN ... 0.0.0.0:8091 ... uvicorn
LISTEN ... 0.0.0.0:9222 ... socat
LISTEN ... [::1]:9222 ... chrome
{"Browser":"Chrome/149.0.7827.55", ... }
{"status":"ok"}
```

---

## 5. systemd 单元文件

### 5.1 xvfb.service

```ini
[Unit]
Description=Xvfb virtual display :99 for headless Chrome
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
ExecStart=/usr/bin/Xvfb :99 -screen 0 1920x1080x24 -nolisten tcp -ac
Restart=on-failure
RestartSec=2

[Install]
WantedBy=multi-user.target
```

### 5.2 chrome-remote.service

```ini
[Unit]
Description=Chrome for Testing with CDP remote debugging on port 9222 (loopback)
After=xvfb.service network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
Environment=DISPLAY=:99
Environment=PULSE_SERVER=disabled
WorkingDirectory=/root
ExecStart=/root/.cache/ms-playwright/chromium-1228/chrome-linux64/chrome \
  --no-sandbox \
  --remote-debugging-port=9222 \
  --user-data-dir=/opt/chrome-remote \
  --no-first-run \
  --disable-gpu \
  --disable-dev-shm-usage \
  --remote-allow-origins=*
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 5.3 chrome-cdp-forward.service

```ini
[Unit]
Description=socat IPv4 forward 0.0.0.0:9222 → [::1]:9222 (Chrome DevTools Protocol)
After=chrome-remote.service network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
ExecStart=/usr/bin/socat \
  TCP4-LISTEN:9222,fork,reuseaddr,bind=0.0.0.0 \
  TCP:[::1]:9222
Restart=on-failure
RestartSec=2

[Install]
WantedBy=multi-user.target
```

### 5.4 mediacrawler-api.service

```ini
[Unit]
Description=MediaCrawler WebUI API (uvicorn)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/MediaCrawler
Environment="PATH=/root/.pyenv/versions/3.11.8/bin:/usr/local/bin:/usr/bin:/bin"
ExecStart=/root/.pyenv/versions/3.11.8/bin/uv run uvicorn api.main:app --host 0.0.0.0 --port 8091 --workers 1
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

---

## 6. 客户端使用方式

### 6.1 Swagger UI（最直观）

浏览器打开：http://192.168.20.173:8091/docs

### 6.2 直接 curl 调 API

```bash
# 启动爬取
curl -X POST http://192.168.20.173:8091/api/crawler/start \
  -H "Content-Type: application/json" \
  -d '{"platform":"xhs","login_type":"qrcode","crawler_type":"search"}'

# 查询状态
curl http://192.168.20.173:8091/api/crawler/status

# 拉日志
curl http://192.168.20.173:8091/api/crawler/logs

# 列出已采集的数据文件
curl http://192.168.20.173:8091/api/data/files

# 下载某个文件
curl -OJ http://192.168.20.173:8091/api/data/download/data/xhs/search/...jsonl
```

### 6.3 实时日志（WebSocket）

```bash
wscat -c ws://192.168.20.173:8091/api/ws/logs
# 或 Python
python3 -c "
import websocket, json
ws = websocket.create_connection('ws://192.168.20.173:8091/api/ws/logs')
while True:
    msg = ws.recv()
    print(json.loads(msg))
"
```

### 6.4 直连 Chrome CDP（绕过 MediaCrawler）

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp("ws://192.168.20.173:9222")
    ctx = browser.contexts[0]            # 复用现有 context（保留登录态）
    page = ctx.new_page()
    page.goto("https://www.xiaohongshu.com")
    # 任意 Playwright API
```

或 curl 探测：

```bash
curl http://192.168.20.173:9222/json/version
curl http://192.168.20.173:9222/json/list           # 所有打开的 Page
```

---

## 7. 关键决策记录

### 7.1 端口选择

- **8091**：MediaCrawler API。原本想用 8080 但被 vclaw control-plane-s 占用，改 8091。
- **9222**：Chrome CDP 默认端口。

### 7.2 Chrome 149 的绑定行为

Chrome 149 **只接受 `[::1]:9222`**，忽略 `--remote-debugging-address=0.0.0.0`（该 flag 已被移除）。

所以链路是：

```
外部 IPv4 客户端 → socat(0.0.0.0:9222) → Chrome([::1]:9222)
```

### 7.3 用 Chrome for Testing 而非系统 Chromium

`snap chromium` 受 AppArmor 限制，既不能绑 0.0.0.0，也无法稳定写入非 snap 标准路径的 user-data-dir。改用 Playwright 自带的 Chrome for Testing 149（普通 ELF 二进制，无沙盒）。

### 7.4 用 Xvfb 而非 --headless

`--headless=new` 会暴露 `navigator.webdriver=true` 等反检测可识别的特征。Xvfb 给 Chrome 一个完整的虚拟 X11 显示，浏览器行为与真实桌面一致，更难被风控识别。

### 7.5 CDP_HOST = "localhost"

MediaCrawler 跑在 192.168.20.173，Chrome 也在同一台机器（绑 `[::1]:9222`）。MediaCrawler 走 loopback 连 Chrome。其他 .20 段机器连的是 **uvicorn API 或 socat 转发后的 CDP**，不走 CDP_HOST。

---

## 8. ⚠️ CDP 确认弹窗

Chrome 144+ 新增：**首次有客户端 attach 时会弹"Allow / Cancel"确认框**，弹在 Chrome 所在机器的屏幕上。

本机 192.168.20.173 当前**无图形登录会话**，所以这个确认框**没人能看到也没人点**。

绕开方法：

1. **临时**：在 MediaCrawler WebUI 里走 `lt=cookie` 而不是 `lt=qrcode`，跳过新登录流程
2. **治本**：用 `chrome-remote-desktop` 或 VNC 在 192.168.20.173 上开图形会话，确认框出现时手动点 Allow
3. **切换模式**：在 `config/base_config.py` 把 `ENABLE_CDP_MODE = False`，MediaCrawler 启动自己的 Playwright 浏览器（不会弹确认框，但失去"复用用户真实浏览器"的反检测优势）

---

## 9. 维护命令

```bash
# 查看状态
systemctl status xvfb.service chrome-remote.service \
  chrome-cdp-forward.service mediacrawler-api.service

# 重启链路（按依赖顺序）
systemctl restart mediacrawler-api.service
systemctl restart chrome-cdp-forward.service
systemctl restart chrome-remote.service
systemctl restart xvfb.service

# 查看日志
journalctl -u mediacrawler-api -f
journalctl -u chrome-remote -f
journalctl -u chrome-cdp-forward -f
journalctl -u xvfb -f

# 清理 Chrome 数据（重置登录态）
systemctl stop chrome-remote.service
rm -rf /opt/chrome-remote/*
systemctl start chrome-remote.service

# 端口占用排查
ss -tlnp | grep -E ':(8091|9222)'
lsof -i :9222
```

---

## 10. 排错速查

| 现象 | 排查 |
|---|---|
| `curl http://192.168.20.173:8091/api/health` 无响应 | `systemctl status mediacrawler-api`；`journalctl -u mediacrawler-api -n 50` |
| `curl http://192.168.20.173:9222/json/version` 无响应 | `ss -tlnp \| grep 9222`；三服务都必须 active；`journalctl -u chrome-remote -n 30` |
| Chrome 启动后立刻死 | Xvfb 没起来；检查 `/tmp/.X99-lock` 是否有残留 |
| socat 报 "Address already in use" | 旧 socat 没死；`pkill -9 socat && systemctl restart chrome-cdp-forward.service` |
| `connect_over_cdp` 连上但 60 秒超时 | Chrome 弹了确认框没人点（见 §8） |
| 数据文件下载 404 | `SAVE_DATA_OPTION` 配置的存储路径在 API 视角下不可达 |

---

## 11. 回滚

```bash
systemctl disable --now \
  xvfb.service \
  chrome-remote.service \
  chrome-cdp-forward.service \
  mediacrawler-api.service
rm /etc/systemd/system/{xvfb,chrome-remote,chrome-cdp-forward,mediacrawler-api}.service
systemctl daemon-reload
rm -rf /opt/chrome-remote   # 可选
```