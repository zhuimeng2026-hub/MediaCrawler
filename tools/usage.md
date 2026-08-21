# MediaCrawler 运行方案对比

> 本文档记录对 `MediaCrawler` 三种运行方案的分析结论。
> 内容基于对源码（`config/base_config.py`、`tools/cdp_browser.py`、`tools/browser_launcher.py`、`cmd_arg/arg.py`、`media_platform/xhs/core.py`）的实际阅读，而非 `README.md` 的逐字搬运。

## 背景

`README.md` 只给出了**默认 CDP 模式**与**关闭 CDP 切换标准 Playwright** 的提示（位于"Chrome 浏览器配置"小节），并未系统说明：除 CDP + Playwright 二选一外，还可以叠加不同的**登录方式**（QR Code / Phone / Cookie）形成多种组合方案。下文把它们拆为 A/B/C 三种实验可跑的方案，便于在不同环境（无桌面、有桌面、有外部浏览器）下选用。

源码关键引用：
- `config/base_config.py:59` — `ENABLE_CDP_MODE = True`（默认）
- `config/base_config.py:83` — `CDP_CONNECT_EXISTING = True`
- `cmd_arg/arg.py:52-57` — `LoginTypeEnum { QRCODE, PHONE, COOKIE }`
- `tools/cdp_browser.py` — CDP 模式入口
- `media_platform/xhs/core.py:34` — `XiaoHongShuCrawler` 引用 `CDPBrowserManager`

## 方案 A：关闭 CDP，纯 Playwright 直跑

**适用环境**：没有图形桌面、没有已登录 Chrome、想快速验证链路能否通到登录页。

**做法**：

```bash
# 1. 关闭 CDP 模式
sed -i 's/ENABLE_CDP_MODE = True/ENABLE_CDP_MODE = False/' \
  /opt/MediaCrawler/config/base_config.py

# 2. 准备好自己的 cookie（最稳），或先走二维码流程
/opt/MediaCrawler/.venv/bin/python main.py \
  --platform xhs \
  --lt cookie \
  --cookies "你自己的 a1 / web_session / webId ..." \
  --type search \
  --headless true \
  --keywords "测试" \
  --crawler_max_notes_count 1
```

**链路验证**：
- 浏览器：使用 `uv sync` 安装时已下载到 `/root/.cache/ms-playwright/chromium-1228` 与 `chromium_headless_shell-1228`，不需要外部 Chrome。
- 进程：实测可见 `chrome-headless-shell ... --user-data-dir=/opt/MediaCrawler/browser_data/xhs_user_data_dir` 已被 Playwright 拉起。
- 用户数据目录：`/opt/MediaCrawler/browser_data/xhs_user_data_dir` 由 `USER_DATA_DIR = "%s_user_data_dir"` 模板生成，首次登录后会被持久化。
- 上限：`config/base_config.py:50` 默认 `HEADLESS = False`，命令行 `--headless true` 会同时覆盖 `HEADLESS` 与 `CDP_HEADLESS`（`cmd_arg/arg.py:357`）。

**风险**：
- 没有 Cookie 时只能停在 QR 登录页，且 `headless=true` 下二维码截图调试不够友好。
- `ENABLE_CDP_MODE = False` 后跳过 `BrowserLauncher` 路径，不会再尝试自动探测本地 Chrome/Edge。

## 方案 B：自己拉一个带调试端口的 Chromium，让 MediaCrawler attach

**适用环境**：服务器端已经装了系统级 Chromium / Chrome，希望复用其登录态与扩展，但不想动 `ENABLE_CDP_MODE`。

**做法**：

```bash
# 1. 启动系统 chromium 并暴露 9222
/usr/bin/chromium-browser --headless=new --no-sandbox \
  --remote-debugging-port=9222 \
  --user-data-dir=/tmp/chrome-debug-profile &

# 2. 验证端口可达
curl -s http://127.0.0.1:9222/json/version | jq .Browser

# 3. MediaCrawler 保持默认配置（ENABLE_CDP_MODE=True, CDP_CONNECT_EXISTING=True）
/opt/MediaCrawler/.venv/bin/python main.py \
  --platform xhs --lt qrcode --type search
```

**链路依据**：
- `tools/cdp_browser.py:152-172` 的 `connect_to_existing_browser()` 优先探测 9222，超时 60s 后才回落到 `BrowserLauncher`。
- `tools/browser_launcher.py` 负责回落到本地 Chrome / Edge 的二进制探测与启动。
- 想强制总是新起而不 attach 现有浏览器，把 `CDP_CONNECT_EXISTING = True` 改成 `False` 即可。

**风险**：
- chromium `--headless=new` 在某些反检测场景不如真实有头浏览器；本仓库注释也提到"某些反检测功能在无头模式下可能无法正常工作"（`base_config.py:73`）。
- 多用户共享 9222 端口会冲突；`config/base_config.py:63` 通过 `CDP_DEBUG_PORT` 调整。

## 方案 C：QR 码扫码登录（headless=false，需要真人介入）

**适用环境**：本地有图形或 X11 转发，第一时间想保存一份真实登录态到 `browser_data/`，供方案 A 之后复用。

**做法**：

```bash
/opt/MediaCrawler/.venv/bin/python main.py \
  --platform xhs \
  --lt qrcode \
  --type search \
  --headless false
```

扫码登录成功后，`SAVE_LOGIN_STATE = True`（默认）会把 `browser_data/xhs_user_data_dir/` 落盘。下次启动方案 A/B 会自动复用。

**链路依据**：
- `cmd_arg/arg.py:169-176` 把 `--lt qrcode` 映射到 `LOGIN_TYPE = "qrcode"`。
- `media_platform/xhs/login.py` 内根据 `LOGIN_TYPE` 分流到 QR / Phone / Cookie 三条处理函数。

**风险**：
- 服务器无 X 时只能 headless，但 headless=true + qrcode 组合下截图获取二维码需要额外脚本（README 未提供）。
- 一次扫码大约维持 7-30 天有效期；过期要重新扫码。

## 选型决策表

| 条件 | 推荐方案 | 关键参数 |
|---|---|---|
| 仅验证安装是否跑得起来 | A | `--headless true`，无需 Cookie |
| 服务器有 cookie 想立刻抓数据 | A | `--lt cookie --cookies "..." --headless true` |
| 服务器无 cookie 但有系统 chromium | B | 外部 chromium 加 `--remote-debugging-port=9222` |
| 本地有 GUI 想首次登录保存状态 | C | `--headless false --lt qrcode` |
| 想最大反检测（用户 Cookie、扩展） | B + C 组合 | C 先扫码落盘，B 再 attach 同一 profile |

## 与 README 的差异

| 主题 | README 表述 | 本文档补充 |
|---|---|---|
| 关闭 CDP 模式 | "在 `base_config.py` 中设置 `ENABLE_CDP_MODE = False`" | 给出完整可执行的 sed + 启动命令模板 |
| CDP 端口 | "页面显示 `Server running at: 127.0.0.1:9222` 表示已就绪" | 补充 `curl http://127.0.0.1:9222/json/version` 验证方式 |
| 登录方式 | 仅在示例中演示 `qrcode` | 显式枚举 `LoginTypeEnum` 的三种取值与适用边界 |
| 浏览器路径 | 只提到本地 Chrome / Edge 自动检测 | 方案 A 给出完全离线（仅 Playwright 自带 Chromium）的回退路径 |

## 复现记录

- 仓库：`/opt/MediaCrawler/`（克隆自 `NanmiCoder/MediaCrawler`）
- Python：`/opt/MediaCrawler/.venv/bin/python` 指向 `Python 3.11.11`
- Playwright 浏览器：`/root/.cache/ms-playwright/chromium-1228` + `chromium_headless_shell-1228`
- 测试时间：MediaCrawler 安装与首跑见 git 历史；方案 A 实测命令见 `usage.md` 顶部代码块。

> 仅供学习与研究目的，禁止商业用途与大规模抓取。详见 `README.md` 顶部免责声明与 `LICENSE` 文件。
