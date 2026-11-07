# dsh-shutdown-guard-plus

> **A fork of [ltayshark0637/dsh-Shutdown-Guard](https://github.com/ltayshark0637/dsh-Shutdown-Guard), not an upstream release.**
> Based on upstream **v1.6.0** (MIT). This fork only adds features; every upstream gate is left untouched.

Auto shutdown for DeepSeekRuntime (DSH) once the task is done and every session is idle, driven by a sidebar guard card with six states.

What this fork adds on top of upstream:

| Addition | Default | What it does |
| --- | --- | --- |
| **Full-screen countdown** | **on** | When the countdown starts, a black overlay shows the remaining seconds in huge type; `Esc` or the on-screen button cancels it |
| **Auto guard** | **off** | Arm automatically at the end of every turn, so you do not have to click "Arm" each time |
| Both are settings rows | — | Under *Settings → dsh-shutdown-guard-plus → Guard*, persisted in the profile |

## Install

```powershell
& "$env:LOCALAPPDATA\Programs\DeepSeekRuntime\resources\runtime\cli\bin\dsh.cmd" plugin --profile desktop add "github:Jd44chs/dsh-Shutdown-Guard-plus"
```

Restart DSH afterwards — `lib/client.js` is only bundled into the client module graph at startup.

## Settings

```
Settings → Plugins → dsh-shutdown-guard-plus → "Guard" tab
```

The first two rows (`Auto guard`, `Full-screen countdown`) are new; the rest are upstream.

## Details

- `fullscreen: boolean` (default `true`) — display only. It never takes part in any gate; turning it off falls back to the sidebar card alone.
- `autoArm: boolean` (default `false`) — when a root agent goes `running → idle`, if `autoArm` is on, nothing is armed yet and `rootsRunning() === 0`, the plugin logs `log.autoArmed` and calls `setArmed(true)`. Everything after that is upstream's five gates, so "background jobs running" and "this turn errored" still block the shutdown. While a countdown runs, the next `running` event still calls `cancelShutdown(false)`.
- Both fields live in the profile (`data/config.json`) and are **not** in `PROFILE_LOCKED_FIELDS`, so they can be changed even on the locked default profile.
- `GET /api/dsh-shutdown-guard-plus/state` exposes both fields to the sidebar card and the floating window.

## Why `autoArm` defaults to off

It really shuts the machine down. With it on, chatting for two minutes and walking away ends with a shutdown 60 seconds later. Turn it on when you want it.

## Plugin id

This fork registers as `dsh-shutdown-guard-plus`, so it can coexist with upstream. Running both at once means two guards watching the same idle sessions — keep only one enabled.

## License

MIT — original copyright ltayshark0637, additions under the same license.

Upstream: [github.com/ltayshark0637/dsh-Shutdown-Guard](https://github.com/ltayshark0637/dsh-Shutdown-Guard)
