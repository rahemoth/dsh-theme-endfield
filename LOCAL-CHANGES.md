# 本地改动（相对上游 1.2.3）

本目录是 `dsh-theme-endfield@1.2.3` 的本地 fork，只加了一个功能：

## 顶部余额胶囊：OpenCode Go 套餐模式

**行为**：胶囊跟随当前会话的模型 provider 自动切换。

| 当前会话模型 provider | 胶囊显示 |
|---|---|
| `opencode-go` | `5h 剩余62%` + `重置1:01:01`（5h 窗口的剩余额度与重置倒计时，圆环同步走剩余份额） |
| 其他 provider | 原行为：DeepSeek 余额 + 峰谷时段；装了 `dsh-codearts-auth` 时仍是渠道额度模式 |

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
  - `ocgRefresh()`：读当前会话的 model directory（与渠道额度模式同一条路径），
    provider 为 `opencode-go` 时进入 OCG 模式；
  - `ocgApply()`：60s 节流取 `/theme-endfield/ocg`，失败 30s 后重试；
  - `ocgPaint()`：只画 5h 窗口——大数字 = 剩余百分比，中段 = 重置倒计时，圆环 = 剩余份额；
  - 离开 OCG 模式时用缓存的最后一次钱包读数还原，不闪 `--`。

**语义约定**：接口的 `percent` 是**已用**比例（与鲸鱼挂件一致），胶囊显示的是
`100 - usedPct`（剩余）。失败画 `--` 而不是 `0`：0 会被误读成"额度用完了"。

## 安装 / 回退

```powershell
# 安装为本地 link（desktop profile）
cd C:\Users\19516\.dsh\profiles\desktop
pnpm remove dsh-theme-endfield
pnpm add link:R:/github/dsh-theme-endfield
# 重启 DeepSeek Harness 桌面端（宿主端插件只在启动时加载）

# 回退到 npm 版
cd C:\Users\19516\.dsh\profiles\desktop
pnpm remove dsh-theme-endfield
pnpm add dsh-theme-endfield@1.2.3
```

## 验证

- 宿主逻辑：`node --test` 之外的独立脚本（mock fetch）覆盖成功归一化 / 无密钥 / 401 不重试 /
  500 重试一次 / 非法 JSON / 结构异常六种情况，全部通过。
- 客户端逻辑：把 `ocg*` 代码块抽出来配 DOM stub 跑过：进入模式、取数上色、节流、
  倒计时、离开模式还原钱包、迟到响应不覆盖、失败画 `--` 并提前重试。
- 真实界面需重启桌面端后在 OpenCode Go 模型会话里观察（浏览器契约无法在 node 里跑）。

## 已知边界

- 只适配了 5h 窗口；`周`/`月` 两个窗口在宿主响应里已带上，胶囊未显示（可在 `ocgPickWindow`
  与 `OCG_PRIMARY_WINDOW` 处改）。
- provider 判定用 `current.provider === 'opencode-go'`（或以其开头），与 DSH
  `llm-pi-ai` 配置里的 provider id 一致。
