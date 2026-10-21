# 倒计时到点的去重口径（按回合，不按时间窗）

> 2026-10-21 立，v1.2.28 起生效。修的是「倒计时到了会连响两次」。
> 号跳到 1.2.28 而不是 1.2.27：versionCode 147 在 2026-10-20 已经带着「爬楼休息统计」入过 lanpub 仓，
> 同一个 versionCode 不能代表两个不同的二进制，否则手机上已经是 147 的人永远收不到这次修复。

## 一次到点有三条投递路径

同一个「倒计时结束」事件会被投到 `Reminder.fire` 三次：

1. 前台服务进程内定时器：`CountdownService.onStartCommand` 里 `handler.postDelayed(this::fire, end - now)`
2. 页面 200ms 轮询归零：`tick()` / `hiitTick()` / `stairRestTick()` → `Native.fire(...)` → JS 桥 → post 到主线程
3. 系统精确闹钟兜底（前台服务被 ROM 杀掉时靠它）：`CountdownScheduler` → `CountdownReceiver`

设计上只该响一次，所以必须有去重。

## 旧口径为什么会漏

旧版是一个进程内 30 秒全局时间窗（`sLastFireAt`），并且 `CountdownService.start()` 在每次登记新一轮倒计时时调 `Reminder.resetDedup()` 把这个窗口清零——本意是别把新一轮的提醒误吞掉。

问题出在两条路径不在同一个线程上收尾：`@JavascriptInterface scheduleCountdown` 跑在 JS 桥线程，清零当场生效；`fire` 是 post 到主线程队列的。HIIT 与爬楼的阶段切换是「先提醒，紧接着登记下一阶段」（`hiitFirePhase()` 后一行就是 `Native.scheduleCountdown(...)`），于是顺序变成：

```
主线程: fire#1 → 记下 sLastFireAt
桥线程: 下一轮 start() → 清零
主线程: fire#2（本轮排队的第二个投递）→ 闸门是开的 → 再响一次
```

两遍音色还不一样：服务路径 `soundType=0`（全局铃声），页面路径带按场景音型。另一个独立成因是兜底闹钟：`setAndAllowWhileIdle` 在 Doze 下会晚投递，而已经派发出去的广播 `AlarmManager.cancel()` 收不回——晚过 30 秒就当成新一轮再响一遍。

## 新口径：回合标识 = 本轮结束时刻

`Reminder.fire(ctx, rest, isTest, soundType, roundEndAt)` 里 `roundEndAt` 是这轮倒计时的结束时刻（毫秒）。同一个 `roundEndAt` 只提醒一次；新一轮自带新的 `roundEndAt`，所以：

- 不需要「登记新一轮时清标记」，`resetDedup()` 整个删除——「怕误吞」和「放过重复」本来就是同一个开关造成的
- 迟到多久都能认出来（不再依赖 30 秒窗口）
- 整段判定在 `synchronized (FIRE_GUARD)` 内，检查与落账之间没有缝

三条路径拿到的必须是同一个数：

| 路径 | 回合标识来源 |
| --- | --- |
| 页面归零提醒 | `state.cdRoundEnd`（`startTimer()` 里算好并同时发给原生，[:741]）；HIIT 用 `hiit.phaseEndAt`，爬楼休息用 `stair.restEndAt` |
| 前台服务 | `CountdownService.endAtMillis`（来自 `EXTRA_END`） |
| 兜底闹钟 | `CountdownScheduler.pending()` 里 `putExtra(EXTRA_END, ...)`，`CountdownReceiver` 读出后传入 |

`pending()` 顺带补上了 `EXTRA_REST`——此前闹钟 Intent 从来没塞过这个 extra，所以走闹钟路径的弹窗一直显示「休息 0 秒」。PendingIntent 的身份只看 action/组件不看 extras，所以 `cancel()` 用空 extras 照样命中同一个闹钟。

已提醒过的回合写进 `fitness_prefs` 的 `fired_round_end_at`：进程被 ROM 杀掉后静态变量归零，迟到的旧闹钟补投仍然认得。

没有回合标识的提醒（HIIT 的「训练完成」，`roundEndAt=0`）退回 3 秒短窗兜底。测试提醒（`test:true`）不进去重。2026-10-21 起爬楼「✓ 结束」不再触发任何提醒——铃声只归倒计时（休息归零那条仍走 `stair.restEndAt` 回合标识）。

## 同一条改动带掉的两处

- 提醒被去重拦下时，悬浮窗横条原来还会再弹一次；现在 `Reminder.fire` 返回是否真提醒，桥侧据此决定要不要 `tryStartOverlay()`
- 页面在倒计时运行中改时长（`setCdMin`/`setCdSec`）不会重新登记原生那一轮，因此让归零提醒复用启动时存下的 `state.cdRoundEnd`，两边对得上，不会各报各的

## 仍然没修的

1. **用哪条路径的音型取决于谁先到**：服务路径不带 `soundType`，基本总是它先到，于是 HIIT/爬楼按场景设的音型实际不生效。要修得把 `soundType` 一起送进倒计时登记，而 HIIT「这一阶段结束该用哪个音型」需要在登记时就推算出下一阶段之后的状态。
2. **通知渠道自带系统默认闹铃音**：Android 8+ 上 `Notification.Builder.setSound(null)` 被忽略，声音由渠道决定。所以 `MediaPlayer` 播自选铃声的同时，通知自己还会播一遍默认闹铃，听感也是「连着响」。修它要把渠道重建为静音，代价是丢掉用户在这条渠道上的手动设置（含绕过勿扰）。
3. 运行中改时长不会重新登记原生那一轮（第 4 节最后一条只保证不会响两遍）。
