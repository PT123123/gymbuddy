# 命名与标识边界

仓库 2026-10-15 由 `fitness-record` 更名为 `gymbuddy`（<https://github.com/PT123123/gymbuddy>，本地 `origin` 已同步，旧地址靠 GitHub 重定向）。
名字只改到"对外称呼"这一层；下面这些标识是**设备上现存训练数据的存储坐标**，改动就等于换了个 App，旧数据读不到。

## 不许改

| 标识 | 位置 | 改了会怎样 |
|---|---|---|
| `com.example.fitness`（namespace + applicationId） | `app/build.gradle:6,20` | 等于新 App：无法覆盖升级，Obtainium 更新链断，沙盒目录换一个，全部数据落在旧目录里读不到 |
| `fitness_records_v5` / `fitness_hiit_v2` / `fitness_stair_v2` / `fitness_notes_v1` | `index.html:603`（`DATA_KEYS`）、`index.html:3134`（`BACKUP_KEYS`） | 备份/恢复只按这四个键名映射 `keys`，改键名后旧备份进来是 `undefined`，直接弹"云端备份里没有可恢复的数据" |
| `files/fitness_data.json` | `FitnessNativeBridge.java:209` | 启动回填（v1.2.19）靠这个文件，改名等于失去落盘镜像 |
| `files/workout_log.jsonl` | `FitnessNativeBridge.java:267,280` | append-only 兜底日志，改名后历史日志与新增日志分家 |
| `fitness_heatmap` / `fitness_prefs` / `fitness_secure` | `WorkoutStore.java:26`、`Reminder.java:35`、`FitnessNativeBridge.java:39` | SharedPreferences 文件改名 = 热力图、提醒设置、加密密钥全部归零（密钥丢了连云端备份都解不开） |
| 通知渠道 `fitness_alarm` | `Reminder.java:28` | 老用户已建的渠道设置失效，强提醒行为回到默认 |

## 可以改（已改）

- 发布资产名：`fitness-record-X.Y.Z.apk` → `gymbuddy-X.Y.Z.apk`，约定写在 README「发布」一节。已发布的 release 不回头改名，Obtainium 只抓最新资产。
- 备份载荷里的 `app:'fitness-record'` → `app:'gymbuddy'`（`index.html:625,2508,3173`）。这个字段**只写不读**：恢复路径判的是 `d.keys`（`index.html:3222`）和 `d.notes`（`index.html:2550`），所以旧的本地快照与云端备份照样能恢复。

## 真要换标识怎么办

先有一份能确认成功的备份（App 内「立即备份到云端」，或导出 JSON 文件），再改代码；恢复侧必须同时接受旧键和新键（旧键→新键映射），否则等于删数据。没有映射就别改。

## 迁数据到新 App（无 root 的唯一通道）

跨包拷贝 `app_webview/`（WebView localStorage）需要 root，设备上没有。走 App 自己的通道：

1. 旧 App：设置 → 立即备份到云端（或导出 JSON），**确认备份时间和组数对得上再往下走**
2. 装新 App
3. 新 App：从云端恢复 / 导入文件

排查通道和事故背景见 `docs/adb-data-recovery.md`、`docs/incident-2026-10-09-midworkout-records-gone.md`。

## 本地目录名

本机目录仍是 `Desktop\fitness-record`，与仓库名不一致是已知状态。自己改名要在**退出 IDE/Qoder 之后**做——工作区根目录被进程当 cwd 占用时，`mv` 报 `Device or resource busy`（目录内的子目录和文件都能正常改名，可用来区分是根目录被占还是权限问题）。改完先 `./gradlew clean` 再构建，`app/build/`、`.gradle/` 的增量缓存里硬编码着旧的绝对路径。
