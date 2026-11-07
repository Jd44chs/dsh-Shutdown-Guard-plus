# 变更记录（本分线）

本文件只记录**分线新增**的改动。上游 v1.6.0 及更早的历史见 [CHANGELOG.upstream.md](CHANGELOG.upstream.md)。

## 1.6.0-plus.1 — 2026-11-07

基于上游 v1.6.0。上游判定逻辑零改动。

### 新增

- **全屏关机倒计时**（`fullscreen`，默认开）
  - `lib/client.js` 新增 `fsEnsure / fsPaint / fsHide / fsBindKeys / fsSync`，由 `setSnapshot()` 末尾的 `fsSync()` 驱动
  - 只在 `state === 'countdown'` 与 `'fired'` 时显示；「取消关机」与 `Esc` 都调上游 `POST /cancel`
  - 整屏遮罩：黑底径向渐变、`clamp(120px,21vw,240px)` 等宽数字、进度条、提示文案
  - 关掉开关后完全不插入遮罩

- **自动守夜**（`autoArm`，默认关）
  - `lib/index.mjs` 的 `ctx.on('agent/status')` idle 分支：`cfg.autoArm === true && !armed && rootsRunning() === 0` → `log('log.autoArmed')` + `setArmed(true)`
  - 之后的判定完全沿用上游五道闸

- **设置页两行开关**
  - 「守夜」标签页新增 `Auto guard` / `Full-screen countdown` 两行，排在 AFK 行之前
  - 走 `profileWrite()` 写入当前 profile，与上游其它字段一致

### 改动

- `lib/index.mjs`
  - `emptyProfile()` 增加 `fullscreen: true`、`autoArm: false`
  - `normalizeProfile()` 保留并归一这两个字段
  - `PROFILE_PATCH_FIELDS` 增加 `fullscreen`、`autoArm`（**不**加入 `PROFILE_LOCKED_FIELDS`）
  - `statePayload()` 下发 `fullscreen`、`autoArm`
- `lib/i18n.mjs`
  - 三语新增 `log.autoArmed`
- `lib/client.js`
  - 三语新增 `set.fs` / `set.fs.desc` / `set.auto` / `set.auto.desc`
  - `fsSync()` 增加 `var allowed = !snap || snap.fullscreen !== false`

### 打包相关

- `package.json`
  - `name` → `dsh-shutdown-guard-plus`
  - `version` → `1.6.0-plus.1`
  - `repository` / `homepage` / `bugs` 指向本分线仓库
  - `scripts.test` 改为三个文件的 `node --check`（上游的 `test/` 与 `scripts/` 目录未随包分发）
- `cordis.patch.yml`
  - 插件 id / name → `dsh-shutdown-guard-plus`
  - config 增加 `fullscreen: true`、`autoArm: false`
