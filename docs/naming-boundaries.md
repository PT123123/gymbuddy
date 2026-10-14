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

## 迁数据到新 App（com.gymbuddy）

跨包拷贝 `app_webview/`（WebView localStorage）需要 root，设备上没有；`run-as` 对非 debug 包也用不了。唯一通道是 App 自己的备份/恢复，而且**四个数据键名没变，所以旧 App 的备份能被新 App 直接认**。

0. **先别卸载旧 App**。现存记录只存在于旧包沙盒里，旧 App 一删就没有第二个来源。迁移核对通过之前它就是你唯一的备份。
1. 旧 App：设置 → 立即备份到云端（或导出 JSON 文件到本机）。**要看得见备份时间和条数再往下走**，只点了按钮没确认成功 = 没备份。
2. 装 `com.gymbuddy` 的 APK（`adb install -r` 或直接装；与旧 App 并存，不覆盖）。
3. 新 App：设置里重新填一次七牛 AK/SK/bucket —— 旧包的加密密钥 `fitness_secure.xml` 在旧沙盒里，新 App 拿不到，不重填连云端备份都解不开。
4. 新 App：从云端恢复 / 导入文件。
5. 核对：记录总条数、最早与最晚日期、笔记条数、HIIT 与爬楼的分段数据。都对上之后再决定要不要删旧 App。

**离线兜底**：`_adb_tmp/`（已 gitignore）里有 2026-10-09 从设备拉下来的 WebView Local Storage leveldb 转储和 `workout_log.jsonl`，可以按 `docs/adb-data-recovery.md` 的办法离线生成一份 `{app,ver,ts,keys}` 形态的 restore JSON 当保底。注意它是 **2026-10-09 的快照**，之后的训练不在里面，只能兜底、不能替代第 1 步。

排查通道和事故背景见 `docs/adb-data-recovery.md`、`docs/incident-2026-10-09-midworkout-records-gone.md`。

## 本地目录名

本机目录仍是 `Desktop\fitness-record`，与仓库名不一致是已知状态。自己改名要在**退出 IDE/Qoder 之后**做——工作区根目录被进程当 cwd 占用时，`mv` 报 `Device or resource busy`（目录内的子目录和文件都能正常改名，可用来区分是根目录被占还是权限问题）。改完先 `./gradlew clean` 再构建，`app/build/`、`.gradle/` 的增量缓存里硬编码着旧的绝对路径。
