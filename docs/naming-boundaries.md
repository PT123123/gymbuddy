# 命名与标识边界

仓库 2026-10-15 由 `fitness-record` 更名为 `gymbuddy`（<https://github.com/PT123123/gymbuddy>，本地 `origin` 已同步，旧地址靠 GitHub 重定向）。
同日把应用包名也换到了 `com.gymbuddy`。**包名换了 = 设备上是一个新 App**；而下面「不许改」那批数据坐标**故意保持原名**——正因为键名没变，旧 App 导出的备份才能原样落进新 App（恢复按 `keys` 里的键名映射）。

## 已改（2026-10-15）

| 标识 | 位置 | 代价 |
|---|---|---|
| `com.example.fitness` → `com.gymbuddy`（namespace + applicationId + 16 个 Java 文件的 package/import + `com/example/fitness/` 目录 + `shortcuts.xml` 的 targetPackage/targetClass + 3 个 Intent action 字符串） | `app/build.gradle:6,20`、`app/src/main/java/com/gymbuddy/`、`res/xml/shortcuts.xml:12,13` | ① 与旧 App 是两个独立应用，**不能覆盖升级**，需手动装一次；② Obtainium 的自动更新链断（它按包名认应用），要把订阅指向新 APK；③ 桌面上旧包的热力图 widget 失效，要重新添加；④ 旧 App 里已排定的精确闹钟/前台服务仍会弹旧 App；⑤ 新 App 是空沙盒，数据要靠备份迁移（见最后一节） |
| Gradle 项目名 `FitnessTracker` → `GymBuddy` | `settings.gradle:14` | 无。只影响构建目录/IDE 模块显示名 |

## 不许改

| 标识 | 位置 | 改了会怎样 |
|---|---|---|
| `fitness_records_v5` / `fitness_hiit_v2` / `fitness_stair_v2` / `fitness_notes_v1` | `index.html:603`（`DATA_KEYS`）、`index.html:3134`（`BACKUP_KEYS`） | 备份/恢复只按这四个键名映射 `keys`，改键名后旧备份进来是 `undefined`，直接弹"云端备份里没有可恢复的数据" |
| `files/fitness_data.json` | `FitnessNativeBridge.java:209`（现路径 `app/src/main/java/com/gymbuddy/`） | 启动回填（v1.2.19）靠这个文件，改名等于失去落盘镜像 |
| `files/workout_log.jsonl` | 同上 `:267,280` | append-only 兜底日志，改名后历史日志与新增日志分家 |
| `fitness_heatmap` / `fitness_prefs` / `fitness_secure` | `WorkoutStore.java:26`、`Reminder.java:35`、`FitnessNativeBridge.java:39` | SharedPreferences 文件改名 = 热力图、提醒设置、加密密钥全部归零（密钥丢了连云端备份都解不开） |
| 通知渠道 `fitness_alarm` | `Reminder.java:28` | 已建的渠道设置失效，强提醒行为回到默认 |

> 这批名字里带 `fitness` 是历史遗留，与包名不一致是**有意的**：改名收益为零，代价是丢数据。

## 可以改（已改）

- 发布资产名：`fitness-record-X.Y.Z.apk` → `gymbuddy-X.Y.Z.apk`，约定写在 README「发布」一节。已发布的 release 不回头改名，Obtainium 只抓最新资产。
- 备份载荷里的 `app:'fitness-record'` → `app:'gymbuddy'`（`index.html:625,2508,3173`）。这个字段**只写不读**：恢复路径判的是 `d.keys`（`index.html:3222`）和 `d.notes`（`index.html:2550`），所以旧的本地快照与云端备份照样能恢复。

## 关于历史文档里的旧包名

`docs/adb-data-recovery.md` 和 `docs/incident-2026-10-09-midworkout-records-gone.md` 里的 `run-as com.example.fitness` 一律**不改**：那是 2026-10-09 事故当时的真实现场和可复现命令，改掉等于篡改取证记录。

## 真要换标识怎么办

先有一份能确认成功的备份（App 内「立即备份到云端」，或导出 JSON 文件），再改代码；恢复侧必须同时接受旧键和新键（旧键→新键映射），否则等于删数据。没有映射就别改。

## 迁数据到新 App：2026-10-15 已经做完

用的不是云端备份那条路，是**整沙盒克隆**：旧包和新包都装的是 debug 包（`debuggable`），`run-as` 对两边都能用，于是把旧包的私有目录原样搬进新包，四个数据键名没变，新 App 一启动就当自己一直是这台设备。

核对结果（设备 `23054RA19C` / Android 15，迁移前后各读一遍）：

| 项 | 旧包 `com.example.fitness` | 新包 `com.gymbuddy` |
|---|---|---|
| 记录条数 | 345 | 345 |
| 覆盖天数 | 19 天（2026-09-12 ~ 2026-10-14） | 同 |
| 笔记 | 22 篇 | 22 篇 |
| 爬楼 | 1 次 | 1 次 |
| HIIT | 无此键 | 无此键 |
| 云端配置 | bucket `fitness-record-android` / z2，最近备份 2026-10-15 成功 | AK、SK 各解出 40 字符，`bkReady()` 成立 |

第 4 步之所以不用重填密钥：`fitness_secure.xml` 里存的是**普通 Base64 的 AES-256 密钥**（`FitnessNativeBridge.java:43 secureKey()`），不是 Android Keystore 里那种不可导出的密钥，所以连着密文（`fitness_backup_cfg_v1` 在 localStorage 里）一起搬过去就能解。

### 命令（按顺序，全程不碰旧包写入）

```bash
adb shell am force-stop com.example.fitness                    # 让 leveldb 落盘、放锁
adb exec-out "run-as com.example.fitness tar -c app_webview"     > app_webview.tar
adb exec-out "run-as com.example.fitness tar -c files shared_prefs" > prefs_files.tar
adb install app/build/outputs/apk/debug/app-debug.apk            # 新包，与旧包并存
adb push app_webview.tar prefs_files.tar /data/local/tmp/ && adb shell chmod 644 /data/local/tmp/*.tar
adb shell "run-as com.gymbuddy sh -c 'cd /data/data/com.gymbuddy && tar -xf /data/local/tmp/app_webview.tar && tar -xf /data/local/tmp/prefs_files.tar && echo OK'"
adb shell rm /data/local/tmp/*.tar                              # 里面有真实训练数据，别留
adb shell pm grant com.gymbuddy android.permission.POST_NOTIFICATIONS
adb shell am start -n com.gymbuddy/.MainActivity
```

`run-as` 解包时属主自动是新的 app uid，`md5sum` 两边比对即可确认没搬坏。**必须先装后解再启动**：先启动一次，WebView 会自己建一个空的 `app_webview/`，再灌 leveldb 就要和新进程抢锁。

验数据落到了新 App，用 DevTools 读页面里的 `state`（debug 包才开着 `setWebContentsDebuggingEnabled`，见 `MainActivity.java:59`）：

```bash
adb forward tcp:9222 localabstract:webview_devtools_remote_$(adb shell pidof com.gymbuddy | tr -d '\r')
```

MIUI 会把后台的 App 冻结，此时 9222 连得上但不吐字节；`am start` 把它唤到前台再读。脚本留在 `_adb_tmp/migrate-20261015/`（已 gitignore）。

新 App 自己会改写 `files/fitness_data.json`（`app` 字段变成 `gymbuddy`）并补写 `workout_log.jsonl`：日志从 145216 字节涨到 203488 字节，是 v1.2.19 启动时把 2026-10-09 上日志功能之前那批存量记录全量入册，属主逻辑在 `index.html:3247 initWorkoutLog()`，按记录 id 去重，不是重复写。

### 另一条通道（正式包 / 没装 debug 包时）

`run-as` 对非 debug 包不可用，只能走 App 自己的备份：

0. **先别卸载旧 App**。现存记录只存在于旧包沙盒里，旧 App 一删就没有第二个来源。
1. 旧 App：设置 → 立即备份到云端。**要看得见备份时间和条数再往下走**，只点了按钮没确认成功 = 没备份。
2. 装 `com.gymbuddy`。
3. 新 App 重填七牛 AK/SK/bucket（这条路带不过去密钥）。
4. 从云端恢复 / 导入文件。
5. 核对条数、最早最晚日期、笔记、HIIT 与爬楼，都对上之后再决定要不要删旧 App。

### 这次没替你做、也替不了的

- **旧 App 还在设备上，没卸载**。核对表就是删它的前提。
- 旧包名下挂着 **10 条待发的一次性倒计时闹钟**（`dumpsys alarm` 里 `com.example.fitness.COUNTDOWN_FIRE`，新包 0 条），它们会照旧弹旧 App 的响铃；卸载旧 App 即清空。
- 桌面热力图小组件是绑包名的，得长按图标 → 添加小组件重新放一次（`fitness_heatmap.xml` 已带过去，重放后热力图直接是完整历史）。
- Obtainium 那边要把订阅换到 `gymbuddy-X.Y.Z.apk` 这个新资产名，旧包的更新链已经断了。

排查通道和事故背景见 `docs/adb-data-recovery.md`、`docs/incident-2026-10-09-midworkout-records-gone.md`。

## 本地目录名

本机目录仍是 `Desktop\fitness-record`，与仓库名不一致是已知状态。自己改名要在**退出 IDE/Qoder 之后**做——工作区根目录被进程当 cwd 占用时，`mv` 报 `Device or resource busy`（目录内的子目录和文件都能正常改名，可用来区分是根目录被占还是权限问题）。改完先 `./gradlew clean` 再构建，`app/build/`、`.gradle/` 的增量缓存里硬编码着旧的绝对路径。
