# 本地改动（相对上游 1.2.3）

本目录是 `dsh-theme-endfield@1.2.3` 的本地 fork，只加了一个功能：

## 顶部余额胶囊：OpenCode Go 套餐模式

**行为**：由设置页新增的「余额来源」下拉决定（宿主字段 `balanceSource`，默认 `auto`）。

| 余额来源 | 胶囊显示 |
|---|---|
| 自动（默认） | 当前会话模型是 `opencode-go` 时显示 OCG，其余显示 DeepSeek 余额 |
| DeepSeek 余额 | 固定原行为：余额 + 峰谷时段（装了 `dsh-codearts-auth` 时仍是渠道额度模式） |
| OpenCode Go 套餐 | 固定显示 5h 窗口：`OpenCode 额度剩余 5h 62%` + `重置1:01:01`，圆环同步走剩余份额 |

OCG 模式下，胶囊的入场动画（brand pose）也换成同一读数：kicker 为 `/// OPENCODE GO`，
标题为 `OpenCode 额度剩余 62%`（取数前为 `--`）；切回余额模式时 kicker 还原为
`/// DEEPSEEK API`，标题由钱包绘制器接管，不会残留 OCG 字样。

OCG 模式下鼠标靠近**不再淡出**：胶囊会长成一张卡片（32px 行 → 126px 卡，走的正是入场
动画那套 height / min-width / padding / `--endfield-balance-cap` 过渡，只是方向反过来，
从"缩小归位"变成"放大成卡"），三个窗口显示在卡片内。头部是居中堆叠的 kicker + 标题，
下面一条细分割线；三行用 34px/auto/1fr 网格对齐，窗口名（`5H`/`周`/`月`）为白色加粗大写字样，
百分比紧跟其后仅 8px，重置时长贴右边缘，卡片左右各留 34px，避免文字贴到圆角端盘：

```
     OpenCode 额度剩余 5h 62%  重置3:12:45  ◐        ← 未悬停
┌──────────────────────────────────────────┐
│            /// OPENCODE GO                │
│               额度剩余                    │
│  ──────────────────────────────────────  │
│  5H     62%                重置 3:12:45   │
│  周     29%                重置 3天5小时  │
│  月     22%                重置 14天19小时│
└──────────────────────────────────────────┘   ← 悬停放大后
```

- 悬停判定用**缓存的胶囊矩形**（挂载、resize、形变结束后各量一次），不是每次 mousemove 都
  `getBoundingClientRect()` —— 形变中的每帧强制布局正是之前"切换卡顿"的来源；展开瞬间还会
  把缓存矩形直接长到目标高度，避免指针往下移进卡片时被误判成离开而反复收起/展开；
- 悬停形变单独用 260ms（入场动画仍是 420ms），并去掉了 padding 过渡（行内容此时不可见），
  少一次逐帧子元素重排；指针事件的展开/收起判定按 rAF 合并，一次手势只写一次状态；
- 行内容与卡片是交叉淡入淡出（复用入场动画的 opacity 过渡），不是硬切；
- 钱包/渠道额度模式保持原来的「靠近淡出」行为，设置页「靠近自动隐藏」开关照旧；
- 卡片三行与行内 5h 读数取自同一份 `/theme-endfield/ocg` 响应，每 1s 刷新剩余与重置时长。

**实现**（两半，都围绕既有 `data-endfield-credit-mode` 属性增加第三种状态 `ocg`）：

- `index.js`（宿主端）
  - `fetchOcgQuota(ctx)`：用 `ctx.credentials.resolve('OPENCODE_GO_API_KEY')` 取密钥，
    `GET https://opencode.ai/zen/go/v1/usage`，`Authorization: Bearer <key>`；
    把 `usage.rolling|weekly|monthly` 归一成
    `{ ok, source:'opencode_go', at, windows:[{key,label,usedPct,remainPct,resetAt}] }`。
    请求逻辑与 dsh-whale-widget 的 `opencode_go` 模板一致（含 5xx 重试一次、4xx 不重试）。
  - `registerOcgBridge(ctx)`：挂 `GET /theme-endfield/ocg`，未配置密钥时返回
    `{ ok:false, why }`（HTTP 200，页面保留上次读数而不是闪错误）。
- `client.js`（浏览器端）
  - 设置页新增「余额来源」行（`balanceSource` select：自动 / DeepSeek 余额 / OpenCode Go 套餐），
    写入后立即重解析并刷新，无需重启页面；
  - `ocgRefresh()`：来源为 `auto` 时读当前会话的 model directory（与渠道额度模式同一条路径），
    provider 为 `opencode-go` 时进入 OCG 模式；来源固定时直接按偏好取舍，不再看 provider；
  - `ocgApply()`：60s 节流取 `/theme-endfield/ocg`，失败 30s 后重试；
  - `ocgPaint()`：只画 5h 窗口——大数字 = 剩余百分比，右端 = 重置倒计时（`margin-left:auto`
    与钱包模式的右侧百分比同位），圆环 = 剩余份额；
  - 离开 OCG 模式时用缓存的最后一次钱包读数还原，不闪 `--`。

**语义约定**：接口的 `percent` 是**已用**比例（与鲸鱼挂件一致），胶囊显示的是
`100 - usedPct`（剩余）。失败画 `--` 而不是 `0`：0 会被误读成"额度用完了"。

## 安装 / 回退

```powershell
# 安装为本地 link（desktop profile）
# 注意：Windows 上不要写 link:R:/...，pnpm 会把盘符 R: 当相对路径，junction 目标会坏掉。
# 正确做法是同盘建一个 junction，再用相对路径 link：profile 与 fork 可以不同盘。
cmd /c mklink /J "%USERPROFILE%\.dsh\profiles\theme-endfield" "R:\github\dsh-theme-endfield"
cd C:\Users\19516\.dsh\profiles\desktop
pnpm remove dsh-theme-endfield
pnpm add link:../theme-endfield
# 重启 DeepSeek Harness 桌面端（宿主端插件只在启动时加载）

# 回退到 npm 版
cd C:\Users\19516\.dsh\profiles\desktop
pnpm remove dsh-theme-endfield
pnpm add dsh-theme-endfield@1.2.3
# 可选：删掉中转 junction（用 rmdir，不会删到 R: 上的真实文件）
cmd /c rmdir "%USERPROFILE%\.dsh\profiles\theme-endfield"
```

## 验证

- 宿主逻辑：`node --test` 之外的独立脚本（mock fetch）覆盖成功归一化 / 无密钥 / 401 不重试 /
  500 重试一次 / 非法 JSON / 结构异常六种情况，全部通过。
- 客户端逻辑：把 `ocg*` 代码块抽出来配 DOM stub 跑过：进入模式、取数上色、节流、
  倒计时、离开模式还原钱包、迟到响应不覆盖、失败画 `--` 并提前重试；
  固定来源（`opencode_go` 无会话也进入 / `deepseek` 立即释放 / `auto` 无会话不猜）也覆盖。
- 悬停展开：单独抽出 hover 几何块测过——带内展开、移到卡片上保持、离开两者收起、
  仅卡片区域不误触发、非 OCG 模式仍走淡出。
- 真实界面需重启桌面端后观察：设置页「余额来源」下拉、以及 OCG 模式下「重置」靠右对齐。

## 已知边界

- 只适配了 5h 窗口；`周`/`月` 两个窗口在宿主响应里已带上，胶囊未显示（可在 `ocgPickWindow`
  与 `OCG_PRIMARY_WINDOW` 处改）。
- provider 判定用 `current.provider === 'opencode-go'`（或以其开头），与 DSH
  `llm-pi-ai` 配置里的 provider id 一致。
