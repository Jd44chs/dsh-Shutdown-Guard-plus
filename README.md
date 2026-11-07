# dsh-shutdown-guard-plus

> **这是 [ltayshark0637/dsh-Shutdown-Guard](https://github.com/ltayshark0637/dsh-Shutdown-Guard) 的衍生分线（fork），不是上游官方版本。**
> 上游基于 **v1.6.0**（MIT）。本分线只做加法，不改上游的任何判定逻辑。

任务结束、会话空闲后自动关机的 DSH 插件。侧边栏一张守夜卡片：武装 / 待命 / 等待后台任务 / 倒计时 / 已下发 / 阻断，六种状态一眼看清。

本分线在上游之上补了三件上游没有的事：

| 新增 | 默认 | 说明 |
| --- | --- | --- |
| **全屏关机倒计时** | **开** | 倒计时一开始，整屏黑底大字显示剩余秒数，按 `Esc` 或点「取消关机」即可撤销 |
| **自动守夜** | **关** | 每轮对话结束后自动进入守夜，空闲满一个倒计时就关机，不用每次手点「武装」 |
| 两个开关都进设置页 | — | 「设置 → dsh-shutdown-guard-plus → 守夜」里可以直接开关，写进 profile，重启后保留 |

---

## 一、上游是什么

[ltayshark0637/dsh-Shutdown-Guard](https://github.com/ltayshark0637/dsh-Shutdown-Guard) 解决的是「让它自己关电脑」这件事：

- **五道判定闸**：已武装 / 无进行中的倒计时 / 全部 root 会话空闲 / 无运行中或停止中的后台任务 / 本批次无会话报错
- **六种状态**：未开启 → 待命 → 等待后台任务 → 倒计时 → 已下发，任意阶段可转阻断
- **AFK 自动接管**：agent 向你提问时自动回「用户处于 AFK，先完成已交代的任务」
- **阻断态分级**：`halted`（额度不足 / 凭证失效 / 上下文超限，需人工点「解除并重试」）与 `waiting`（限流 / 5xx / 网络中断，按重试策略自动重试）
- 默认配置锁死 60 秒倒计时 + 等后台任务 + 出错跳过，改动须新建自定义配置

这些机制本分线**原样保留**。上游的 README 更完整，判定细节请以它为准：[README.md（上游）](README.upstream.md)。

## 二、本分线改了什么

### 1. 全屏关机倒计时（`fullscreen`，默认 `true`）

倒计时开始时，在窗口上覆盖一层整屏遮罩：

- 黑底径向渐变 + 超大等宽数字（`clamp(120px, 21vw, 240px)`）+ 进度条
- 文案：`正在关机` / `任务已完成 · 点按钮或按 Esc 取消`
- 「取消关机」按钮与 `Esc` 都走**上游原有的** `POST /cancel`，不绕过状态机
- 只读 snapshot，不参与任何判定；`state` 一离开 `countdown` / `fired` 立即移除遮罩
- 关掉这个开关后完全不插入遮罩，退化成只用侧边栏卡片

实现位置：`lib/client.js` 的 `fsEnsure / fsPaint / fsHide / fsBindKeys / fsSync`，由 `setSnapshot()` 末尾调用 `fsSync()` 驱动。

### 2. 自动守夜（`autoArm`，默认 `false`）

上游只能手点「武装」。打开这个开关后：

```
agent/status 由 running 变为 idle
        ↓
cfg.autoArm === true 且当前未武装 且 rootsRunning() === 0
        ↓
log('log.autoArmed') → setArmed(true) → 上游原有的 tryArm()
```

也就是说，**武装之后的一切判定仍然是上游那五道闸**。它不会绕过「有后台任务就不关」「本轮报错就不关」。

安全性还靠上游已有的一条规则兜底：倒计时期间只要 agent 再次进入 `running`，`cancelShutdown(false)` 会立刻撤销倒计时 —— 你接着说下一句，就不会被关机。

> **为什么默认关**：这个开关会真的关机。开着它闲聊两句，聊完 60 秒电脑就关了。想用再开。

### 3. 两个开关的存储位置

两者都在 profile 里，和其它字段一样写进 `data/config.json`：

| 字段 | 类型 | 默认 | 是否锁死 |
| --- | --- | --- | --- |
| `fullscreen` | boolean | `true` | 否，默认配置下也能改 |
| `autoArm` | boolean | `false` | 否，默认配置下也能改 |

它们**不在** `PROFILE_LOCKED_FIELDS` 里 —— 上游把 `countdownSec` / `marginSec` / `onlyOnSuccess` / `blockOnJobs` / `afkAutoAnswer` 五个字段锁死成安全基线，这两个显示与便利性开关不属于安全基线，所以允许改。

`GET /api/dsh-shutdown-guard-plus/state` 会下发 `fullscreen` 与 `autoArm`，侧边栏卡片与独立悬浮窗 `/window` 读的是同一份状态。

## 三、安装

有两条路，任选一条：

**① 直接从 GitHub 装（推荐）**

```powershell
& "$env:LOCALAPPDATA\Programs\DeepSeekRuntime\resources\runtime\cli\bin\dsh.cmd" plugin --profile desktop add "github:Jd44chs/dsh-Shutdown-Guard-plus"
```

**② 克隆到本地再装目录**（`github:` 前缀在你的 dsh 版本上不被接受时用这条）

```powershell
git clone https://github.com/Jd44chs/dsh-Shutdown-Guard-plus
& "$env:LOCALAPPDATA\Programs\DeepSeekRuntime\resources\runtime\cli\bin\dsh.cmd" plugin --profile desktop add "E:\path\to\dsh-Shutdown-Guard-plus"
```

装完 `dsh.profile.bundles` 会自动追加 `dsh-shutdown-guard-plus`，无需手工登记。

**装完必须重启 DSH**：`lib/client.js` 只在 DSH 启动时编入客户端模块图，刷新页面不会生效。

### 和上游同装会怎样

本分线把插件 id 改成了 `dsh-shutdown-guard-plus`，所以可以和上游 `dsh-shutdown-guard` 并存。但同时开着两个守夜插件，两个都会盯着「会话空闲」这件事，容易互相干扰。**建议只留一个**。

## 四、设置页在哪

```
设置 → 插件 → dsh-shutdown-guard-plus → 「守夜」标签页
```

从上到下依次是：自动守夜 / 大屏关机倒计时 / 后台任务守卫 / 出错跳过 / 倒计时时长 / AFK 自动回答。前两个是本分线新增的行，其余是上游原有。

## 五、风险与边界

- 本分线只调 Windows 自带的 `shutdown.exe`（`/s /t <sec>` 与 `/a`），不联网、不写系统目录、无 runtime 依赖、无 postinstall。
- `autoArm` 打开后，**每轮对话结束都会进入倒计时**。如果你正在等一个长回答但中途去干别的，可能被关机。觉得吵就关掉它。
- 全屏遮罩覆盖整个 webContents，但 Electron 窗口层左上角的标题徽标在 webContents 之外，压不住（无影响）。
- 不处理休眠 / 待机，只做关机。

## 六、与上游同步

- 基线：上游 `v1.6.0`。
- 上游若有安全修复，合并策略是：**先取上游整份文件，再重放本分线的补丁**（见 `tools/` 下的补丁脚本思路），而不是手工三方合并 —— 因为上游代码是单文件大文件，手合并容易漏。
- 想退回纯上游：把 `lib/` 换回上游版本，删掉 `fullscreen` / `autoArm` 两个字段即可，无需迁移数据。

## 七、许可

MIT，见 [LICENSE](LICENSE)。

- 原版权：Copyright (c) ltayshark0637（上游作者）
- 本分线新增部分同样以 MIT 发布

上游链接：[github.com/ltayshark0637/dsh-Shutdown-Guard](https://github.com/ltayshark0637/dsh-Shutdown-Guard)
