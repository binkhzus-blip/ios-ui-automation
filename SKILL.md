---
name: ios-ui-automation
description: Use when driving an iOS app programmatically — tapping, swiping, typing, reading UI state, or taking screenshots. Triggers include UI automation, exploratory testing, reproducing a user-reported bug, verifying a flow end-to-end, or any "please click X in the app" request. Covers both simulator (IDB preferred) and real device (iOS 17+ via Appium-wrapped WDA). Tool-neutral — IDB / WDA / Appium / simctl as appropriate.
---

# iOS UI Automation

## Overview

Drive an iOS simulator app from the shell: read the screen, tap, swipe, type, screenshot. The **goal** is tool-neutral — pick whichever tool can actually hit the element you want. Coordinates alone (without knowing what's under them) is a last resort.

**Core principle:** Playbook 优先，AX tree 其次，盲坐标最后。

## ⚠ 执行流程（必须严格遵守）

收到 UI 自动化任务时，按以下顺序执行。**不要跳步**。

### Step 0: 检查 Playbook

在项目里查找是否已有自动化 playbook（常见目录名：`automation-playbooks/`、`ui-automation/`、`e2e/` 等）。

```bash
# 查找可能的 playbook 目录
ls automation-playbooks/ 2>/dev/null || ls ui-automation/ 2>/dev/null || ls e2e/ 2>/dev/null
```

如果存在，查看里面的结构：
- **flows/** — 端到端流程（含可直接运行的 bash 脚本）
- **steps/** — 原子操作步骤
- **lib/** — 可复用的 helper 函数库

然后决定走哪条路：
- **有匹配的 flow** → Step 1A（直接跑脚本）
- **有部分 step 但没有完整 flow** → Step 1B（组合已有 step）
- **什么都没有** → Step 1C（从零探索，完成后必须写 playbook）

### Step 1A: 直接执行 Flow 脚本（首选）

Flow 文件通常包含完整的 bash runner 脚本。**直接执行，不要逐步手动跑**。

**执行原则：**

1. **一口气跑完，不要每步截图确认**。只在最后截图验证终态。中间插入截图和 describe-all 会大幅拖慢速度。
2. **如果某步失败**，查看 flow 文件的 "Known failure modes" 部分，按记录的 Fix 处理。
3. **如果 helper 函数失败**（特别是 WebView 相关的），立刻改用 step 文件里记录的固定坐标作为备用方案。

### Step 1B: 组合已有 Step

如果没有完整 flow，但有单个 step 文件：

1. 读取每个相关 step 的可执行命令（通常标为 "Portable invocation"）
2. 按顺序组合成脚本，在步骤间加合适的 `sleep`
3. 一口气执行
4. 完成后 → **写一个新的 flow 文件**沉淀经验

### Step 1C: 从零探索

只有在完全找不到相关 playbook 时才走这条路。

1. 用 Canonical Workflow（describe-all → describe-point → tap → screenshot）逐步探索
2. **完成后必须写 playbook**（step + flow 文件），不能只跑完就结束

### Step 2: 验证结果 & 输出耗时

- 最后截图确认终态
- 如果 flow 文件定义了验证条件（如检查特定 AXLabel），执行该验证
- **必须输出本次执行耗时**。在 flow 脚本开头记录 `start_ts=$(date +%s)`，结束时计算并输出总耗时和各阶段耗时。示例输出格式：
  ```
  ✅ ORDER PLACED: 7140
  ⏱ 总耗时: 72s
  　 Steps 1–15 (原生 UI): 44s
  　 Steps 16–18 (WebView): 17s
  　 Step 19 (等待结果): 11s
  ```
- 如果 flow 文件里有 "Time cost comparison" 表，将本次结果与历史数据对比，报告是否有改善或退步

### Step 3: 更新 Playbook & 优化耗时

- 执行过程中发现新的 gotcha → 更新 step 文件
- 新流程 → 写 flow + step 文件
- 验证了已有坐标仍然有效 → 更新 `last_verified` 日期
- **持续优化执行速度，目标是每次都比上次更快**：
  - 检查每步的 `sleep` 是否过长——能否改成 polling（等指定 label 出现后立即继续）替代固定 sleep
  - 如果某步实际只需要 1s 但 sleep 了 3s，缩短它
  - 合并可以并行的操作（如截图验证和 AX 查询同时做）
  - 将优化后的耗时更新到 flow 文件的 "Time cost comparison" 表中
  - 记录哪些 sleep 被缩短了、哪些步骤被优化了，方便下次继续改进

## Playbook 编写规范

### 目录结构

```
automation-playbooks/
├── _devices/          # 设备配置（UDID、分辨率、scale）
├── flows/             # 端到端流程（含 bash runner 脚本）
├── steps/             # 原子操作步骤
└── lib/               # 可复用 bash 函数库（helpers.sh 等）
```

### Helper 库

项目应有一个 `lib/helpers.sh`（或类似），封装常用操作。推荐封装的函数类型：

| 类别 | 功能 | 备注 |
|---|---|---|
| **环境** | 检查工具链就绪、获取窗口尺寸 | |
| **按 label 点击** | 通过 AXLabel 查找元素并 tap | ✅ 原生元素可靠 |
| **按 label 点击（带重试）** | 页面转场后重试 | ✅ 原生元素可靠 |
| **按 ID 点击** | 通过 AXUniqueId 查找 | ✅ 最稳定 |
| **探测点击** | describe-point 确认后 tap | ✅ overlay/弹窗可靠 |
| **网格点击** | 在聚合 StaticText 内按比例定位 | ✅ 聚合文字可靠 |
| **WebView 按钮** | 扫描 WebView 区域找按钮 | ⚠ 不稳定，需备用坐标 |
| **等待** | 轮询直到指定 label 出现 | |
| **滚动** | 带正确 duration + delta 的 swipe | |
| **截图** | 截图保存到指定路径 | |

### Step 文件格式

每个 step = 一个原子操作（1 次 tap / 1 次 swipe / 1 次输入）。

推荐结构：

```markdown
---
id: step-short-name
device: <device-name>
status: ✅ 或 ⚠ gotcha
verified: <date>
---

# <语义描述>

## Target — 目标元素描述
## Preconditions — 执行前 app 必须处于什么状态
## Portable invocation — 用 helper 函数的一行 bash（不硬编码坐标）
## Verified methods — 表格：工具 / 方法 / 状态 / 坐标
## Gotchas — 只记录耗时 >3 分钟或反直觉的发现
## After state — 点击后预期的状态
```

### Flow 文件格式

每个 flow = step 序列 + 可直接运行的 bash 脚本。

推荐结构：

```markdown
---
flow: <flow-name>
device: <device-name>
last_verified: <date>
status: ✅ end-to-end working
---

# <标题>
## Preconditions — 完整前置条件
## Steps 表格 — 每行链接到 step 文件
## Full bash runner — 可直接复制运行的完整脚本
## Known failure modes — 表格：Symptom / Cause / Fix
```

### 关键约定

- **坐标绑定设备**：所有坐标必须标注对应的设备，不同设备坐标不通用
- **不允许修改 app 业务逻辑**：加 `accessibilityIdentifier` 合法，改默认值/跳过交互不合法
- **Portable invocation 用 helper 函数**：不硬编码绝对坐标

## Tool Preference

| Tool | Simulator | Real device (iOS 17+) | Install |
|---|---|---|---|
| **IDB** | ✅ First choice. `describe-all` + `describe-point` give exact frames via accessibility API | ❌ companion can't pin to real-device UDID | `brew install facebook/fb/idb-companion` + `pip3 install fb-idb` |
| **Appium + WDA** | Fallback. Heavier than IDB | ✅ **First choice for real device.** Wraps WDA with keep-alive / auto-restart — necessary because iOS 17+ jetsam kills backgrounded xctest runners | `npm install -g appium && appium driver install xcuitest` |
| **Raw WDA via `pymobiledevice3 wda`** | n/a | ❌ Don't. xctrunner gets jetsam'd within seconds of launching the app under test → :8100 dies → unrecoverable | n/a |
| **simctl** | ✅ Low-level ops: install/uninstall, URL schemes, push, keychain, privacy | n/a (simulator only) | Built into Xcode |
| **`xcrun devicectl` + `pymobiledevice3`** | n/a | ✅ Install/launch/screenshot/file-pull on real device (no UI tap) | `xcrun devicectl` built-in; `pipx install pymobiledevice3` |

## Prerequisites (IDB path — simulator)

1. App running on simulator.
2. `idb_companion` running, pinned to the right simulator:
   ```bash
   idb_companion --udid <UDID> > /tmp/idb_companion.log 2>&1 &
   # wait for "grpc_port" line, then
   idb connect localhost <port_from_log>
   ```
3. Verify: `idb list-targets | grep <UDID>` shows `localhost:<port>` at the end.

## Prerequisites (Appium path — real device, iOS 17+)

Real-device UI automation requires wrapping WDA in Appium. **Don't try raw `pymobiledevice3 developer wda launch -xc`** — the xctrunner gets jetsam'd within seconds of launching the app under test (iOS 17+ background xctest policy), and the HTTP session never recovers. Appium handles the lifecycle (keep-alive ping, auto-restart on crash).

One-time setup:

1. **Tunneld** (RSD tunnel for iOS 17+):
   ```bash
   sudo pymobiledevice3 remote tunneld &       # runs on :49151
   curl -sS http://127.0.0.1:49151/             # returns {"<UDID>":[{...}]}
   ```
2. **Trust + Developer Mode + unlocked screen** on the device.
3. **Pre-build a signed WDA** in a stable location (e.g. `~/code/WebDriverAgent`). Open in Xcode → assign a team that has a wildcard provisioning profile (a personal Apple ID with `<team>.*` works fine for dev). Build once via `xcodebuild ... test` so DerivedData has `WebDriverAgentRunner-Runner.app`. This is the WDA Appium will reuse.
   - Verify signing with `codesign -dvvv` — note the actual `TeamIdentifier` and `Identifier`; pass these to Appium below.
4. **Install Appium + xcuitest driver**:
   ```bash
   npm install -g appium                     # appium 3.x
   appium driver install xcuitest            # xcuitest 11.x (ships appium-ios-remotexpc for iOS 18+ RemoteXPC)
   appium driver doctor xcuitest             # must show all required ✅
   ```

Per-session:

```bash
# Start server (PATH must include /usr/sbin so lsof is reachable)
PATH="/usr/sbin:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin" \
  appium --address 127.0.0.1 --port 4723 --log /tmp/appium/server.log &

# Create session against the real device + app
curl -sS --noproxy '*' -X POST http://127.0.0.1:4723/session \
  -H 'Content-Type: application/json' \
  -d '{
    "capabilities": {
      "alwaysMatch": {
        "platformName": "iOS",
        "appium:automationName": "XCUITest",
        "appium:udid": "<DEVICE-UDID>",
        "appium:platformVersion": "<iOS-version>",
        "appium:bundleId": "<app-bundle-id>",
        "appium:xcodeOrgId": "<team-from-codesign-dvvv>",
        "appium:xcodeSigningId": "Apple Development",
        "appium:bootstrapPath": "/path/to/your/WebDriverAgent",
        "appium:derivedDataPath": "/path/to/DerivedData/WebDriverAgent-<hash>",
        "appium:noReset": true,
        "appium:newCommandTimeout": 600
      }
    }
  }'
# returns {"value":{"sessionId":"...","capabilities":{...}}}
```

**Why `bootstrapPath` + `derivedDataPath` are critical**: by default Appium uses its own WDA copy under `~/.appium/node_modules/.../appium-webdriveragent/` and invokes `xcodebuild` **without** `-allowProvisioningUpdates`. Any new xctrunner bundle id (e.g. `com.tech.WDA.xctrunner`) has no provisioning profile → `xcodebuild` exits 65. Pointing at a pre-built, pre-signed WDA bypasses the rebuild entirely.

Per-action (W3C actions API):

```bash
SID=<sessionId from session create response>

# Tap at logical (x, y) — NOT screenshot pixels
curl -sS --noproxy '*' -X POST "http://127.0.0.1:4723/session/$SID/actions" \
  -H 'Content-Type: application/json' \
  -d "{\"actions\":[{\"type\":\"pointer\",\"id\":\"f1\",\"parameters\":{\"pointerType\":\"touch\"},\"actions\":[{\"type\":\"pointerMove\",\"duration\":0,\"x\":X,\"y\":Y},{\"type\":\"pointerDown\",\"button\":0},{\"type\":\"pause\",\"duration\":80},{\"type\":\"pointerUp\",\"button\":0}]}]}"

# Screenshot (base64 PNG in .value)
curl -sS --noproxy '*' "http://127.0.0.1:4723/session/$SID/screenshot" \
  | python3 -c "import json,sys,base64; open('/tmp/s.png','wb').write(base64.b64decode(json.load(sys.stdin)['value']))"
```

**Real-device gotchas (in addition to the simulator ones):**

| Gotcha | Fix |
|---|---|
| Local proxy intercepts IPv6 → curl to RSD tunnel returns 502 | Always `--noproxy '*'` on Appium/WDA curls |
| Appium PATH doesn't include `/usr/sbin` → `lsof not found` → can't clean stale ports | `export PATH=/usr/sbin:...` before launching appium |
| WDA stops responding mid-session (system reclaim) | Appium auto-rebuilds; raise `newCommandTimeout` to avoid premature kills |
| First session creation is slow (~30s) for xcodebuild test-without-building handshake | Expected; subsequent sessions are faster (cache hits) |
| Screen sleeps → lockdownd unadvertises → screenshot fails "Device is not connected" | Keep screen unlocked; consider AssistiveTouch tap to wake |

For project-specific recipes (exact paths, team ID, pre-built DerivedData location, app coordinates), check the project's `automation-playbooks/_devices/<device>.md`.

## Core Endpoints (IDB)

| Operation | Command |
|---|---|
| **List elements** | `idb ui describe-all --udid $DEV` — JSON array of AX elements with frames |
| **Hit-test at point** | `idb ui describe-point --udid $DEV X Y` — element at (X,Y) |
| **Tap** | `idb ui tap --udid $DEV --duration 0.1 X Y` — **always use `--duration 0.1`** |
| **Swipe / scroll** | `idb ui swipe --udid $DEV --duration 0.5 --delta 5 X1 Y1 X2 Y2` — **both flags required** |
| **Screenshot** | `idb screenshot --udid $DEV /tmp/s.png` |
| **Type text** | `idb ui text --udid $DEV "hello"` |
| **Launch app** | `idb launch --udid $DEV <bundleId>` |

All commands take **logical point** coordinates, **not screenshot pixels** (which are 2× or 3× on Retina devices).

## Canonical Workflow（仅在无 Playbook 时使用）

> 如果已有 playbook，跳过这一节，直接执行 bash runner。

```
1. describe-all → find target by AXLabel / AXUniqueId / type
2. if found: compute center of frame, tap
3. if NOT found: describe-point at estimated location → confirm element
4. tap that coordinate
5. screenshot to verify state changed
6. 探索完成后 → 把经验写成 playbook（step + flow 文件）
```

## describe-all 的盲区（重要）

`idb ui describe-all` 不是完整的 AX tree，它只返回主窗口顶层可访问元素。经常漏掉：

| 漏掉的情况 | Workaround |
|---|---|
| Modal overlays / 弹窗 | `describe-point` at button location |
| `UITabBarItem`（自定义 tab bar） | `describe-point` at tab bar area |
| WKWebView 内容 | `describe-point`（但不稳定，见下方） |
| 聚合 StaticText（多个子元素合并成一个 AX 元素） | 按比例坐标 tap，或让开发加 identifier |
| `shouldGroupAccessibilityChildren=true` 的子元素 | 同上 |

**Rule of thumb:** 看得到但 describe-all 列不出来 → 先试 `describe-point`。

## WKWebView 的特殊处理

WKWebView 内的元素 **可访问性不稳定**：

- `describe-all` 基本不返回 WebView 内部元素
- `describe-point` 有时能返回 Button/Link 等信息，有时全部返回 null
- **同一个页面、同一个坐标，不同时刻结果可能不同**

**应对策略：**

1. 先尝试 describe-point 扫描 WebView 区域找按钮
2. 如果扫描失败，从截图估算按钮位置，直接 tap 固定坐标
3. **在 playbook 的 step 文件里同时记录两种方法**：helper 调用 + 固定坐标备用
4. bash runner 里优先用固定坐标（更快更稳），helper 留给探索阶段

## AX Label 查询策略

按可靠性排序：

1. `AXUniqueId`（对应 `accessibilityIdentifier`）— 最稳定
2. `AXLabel` 有意义的字符串 — 第二选
3. Type + position（如 "y 在 500-600 之间的唯一 Button"）— jq filter
4. `describe-point` at 截图坐标 — last resort

如果关键元素没有有用的 label/id，**让开发加 `accessibilityIdentifier`**（纯 metadata，不影响行为）。

## Coordinate System

Tap / swipe 用 **logical points**，不是截图像素：

- iPhone 16 Pro: logical `402 × 874`, screenshot `1206 × 2622` (3×)
- iPhone 15 / 14 / 13: logical `390 × 844`
- iPad Pro 13": logical `1032 × 1376`

获取当前设备逻辑分辨率：
```bash
idb ui describe-all --udid $DEV | jq -r '.[] | select(.type=="Application") | "\(.frame.width)x\(.frame.height)"'
```

**坐标和设备绑定**——不同设备的坐标不通用。

## Common Patterns

**按 label 查找并点击：**
```bash
# 用 helper（如果项目有）
tap_label "Check Out"

# 或手动
coords=$(idb ui describe-all --udid $DEV | jq -r '[.[] | select(.AXLabel=="Check Out")] | .[0] | "\(.frame.x + .frame.width/2 | floor) \(.frame.y + .frame.height/2 | floor)"')
idb ui tap --udid $DEV --duration 0.1 $coords
```

**Overlay 弹窗内的按钮（describe-all 看不到）：**
```bash
idb ui describe-point --udid $DEV 200 540   # 确认元素存在
idb ui tap --udid $DEV --duration 0.1 200 540
```

**滚动直到元素可见：**
```bash
for i in 1 2 3; do
  idb ui describe-all --udid $DEV | jq -e '.[] | select(.AXLabel=="Target" and .frame.y >= 0 and .frame.y < 800)' >/dev/null && break
  idb ui swipe --udid $DEV --duration 0.5 --delta 5 200 700 200 200
  sleep 1
done
```

## Recovery Patterns

- **IDB companion hung** → `pkill idb_companion`; restart + reconnect
- **WDA snapshot stalled (simulator)** → `pkill -f "xcodebuild.*WebDriverAgent"`; restart from scratch
- **Appium session died (real device)** → DELETE `/session/<sid>`; create a fresh session. Appium will re-launch WDA via `xcodebuild test-without-building`. If that also fails, check `lsof -i :8100` for stale processes, then re-create.
- **App lost login state** → reinstall may wipe UserDefaults; log in first or inject auth token

## Common Mistakes

| Mistake | Fix |
|---|---|
| **不查 playbook 就开始手动探索** | 先查 flows/ 目录——已有 flow 直接跑脚本 |
| **逐步执行脚本，每步插截图** | 一口气跑完，只在最后验证终态 |
| **WebView 按钮只靠自动扫描** | WKWebView 可访问性不稳定；失败时改用 step 文件记录的固定坐标 |
| 用截图像素坐标 tap | 除以 device scale（@3x 除以 3）得到 logical points |
| `idb ui tap` 不加 `--duration` | 加 `--duration 0.1`，否则某些按钮不响应 |
| `idb ui swipe` 不加 `--duration` / `--delta` | 必须两个都加，否则 ScrollView 不识别 pan |
| 只信 describe-all | tab bar、overlay、WebView 都可能漏；用 describe-point 二次确认 |
| `AXLabel` 是 "Button" 或 null 时按 label 查 | 改用位置或 AXUniqueId；让开发加 identifier |
| 修改 app 代码跳过弹窗 | 用 describe-point 操作弹窗，保持真实行为 |
| **跑完新流程不写 playbook** | 必须沉淀为 step + flow 文件 |

## When NOT to Use

- Unit tests — use XCTest directly.
- CI regression — use XCUITest / Appium with structured test cases.
- Throughput-critical automated test suites — this skill is for exploratory / one-off / bug-repro work, not for sustained CI load.
