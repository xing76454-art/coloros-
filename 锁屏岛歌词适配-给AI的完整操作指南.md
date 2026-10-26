# ColorOS 锁屏岛歌词适配 —— 给 AI 的完整操作指南

> 目标读者：另一个 AI / 开发者，希望把**任意第三方音乐播放器**接入 **OPPO / 一加 / realme（ColorOS 16+）锁屏岛歌词**。
> 本文是一次真实成功的完整复盘（以「Melody-Domain」播放器接入为例，真机 ColorOS 16 验证通过），
> 覆盖：**机制原理 → 数据契约 → 逐条操作步骤 → 工具清单 → 调试方法 → 踩坑经验**。
> 照着做即可复现，无需从头逆向。

---

## 0. 一句话结论（先建立正确预期）

锁屏岛歌词 **不需要 root，不依赖任何 Xposed 组件**。播放器只要在自己的
`MediaSession` 的 `MediaMetadata` 里发布一个 JSON 键 **`lyricInfo`**（外加几个兼容键），
ColorOS 系统锁屏岛就会**原生消费**并渲染歌词。进度不需要播放器自己刷新 metadata，
系统读 `PlaybackState` 的时钟即可。

> 需要 root 的只有「高级视觉效果」（逐字逐帧动画、翻译按钮、亮度字号调节等），
> 那属于 Bridge（Xposed 模块）的增强，与基础歌词显示无关。

---

## 1. 原理：数据流与系统准入链

### 1.1 完整链路

```text
播放器 MediaSession.setMetadata(...)
        │
        ▼
system_server: com.android.server.media.OplusMediaControlService
   ├─ isInHistoryPlayInfoWhiteList(pkg)      ← 媒体历史白名单（按包名）
   ├─ isInMediaBlackList(pkg)                ← 媒体黑名单（按包名；被拉黑则全断）
        │
        ▼
SystemUI: OplusMediaDataManagerExImpl.onMetaDataChanged(pkg, metadata)
   ├─ getRusWhiteList()                       ← RUS 云端白名单（包名 List）
   ├─ getLyricEntrance(pkg):int               ← 歌词入口等级
   ├─ resolveLyricSupport(pkg, metadata)      ← 关键门控（见 1.2）
   ├─ loadLyricInBg(pkg, metadata) → metadata.getString("lyricInfo")  ← 读歌词 JSON
        │
        ▼
MediaLiveAlertController 渲染锁屏岛（进度取 PlaybackState 时钟）
```

### 1.2 关键门控：`resolveLyricSupport`（最容易被忽略的一环）

```text
getLyricEnable(pkg) != 1  → 放行（不需要握手）
getLyricEnable(pkg) == 1  → 必须 metadata.getString("oplus.media.config") == "1"（握手！）
```

- `getLyricEnable` 是 **RUS 云端下发**的配置，最新系统可能已对更多包置 1。
- 因此**永远无条件发布 `oplus.media.config="1"` 这个握手键**——系统忽略它时无副作用，
  需要它时能解锁门控。这是「旧版能显示、新版调不出来」的经典原因。

### 1.3 包名验证的真相

- 系统侧白名单**只按包名放行，不校验签名**。
- 证据：网易云签名与欢太音乐签名完全不同，但两者都能显示 → 说明校验口径是包名。
- **结论**：把播放器包名改成白名单内某知名音乐 app 的包名即可通过（本文用 `com.netease.cloudmusic`）。

---

## 2. 数据契约（发布什么、什么格式）

### 2.1 核心键位表

| 键 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `lyricInfo` | String(JSON) | ✅ 核心 | 锁屏歌词页的主数据源，格式见 2.2 |
| `oplus.media.config` | String `"1"` | ✅ 建议 | RUS lyricEnable 门控握手（无副作用） |
| `ucar.media.metadata.LYRICS_WHOLE` | String | ✅ 建议 | 整篇逐行 LRC（车机/媒体管线兼容键） |
| `ucar.media.metadata.LYRICS_STATUS` | Long | ✅ 建议 | `101`=有歌词，`102`=无歌词 |
| `ucar.media.metadata.LYRICS_LINE` | String | ⚠️ 可选 | 当前正在唱的**那一行**，随播放逐行更新 |

> `LYRICS_LINE`（当前行）是厂商（欢太音乐）逐行更新用来驱动锁屏岛实时跳行的，
> 但**只要 `lyricInfo` JSON 里有完整时间轴 + PlaybackState 时钟正常，系统就能自己算当前行**，
> 所以基础显示**不强制要求**发 `LYRICS_LINE`。

### 2.2 `lyricInfo` JSON 结构（可直接照抄）

```json
{
  "songName": "歌曲名",
  "artist": "歌手",
  "songId": "123456",
  "lyricType": 0,
  "lyric": "[00:00.000]第一句歌词\n[00:05.240]第二句歌词\n",
  "rawLyric": "[00:00.000]<00:00.000>逐<00:00.500>字<00:01.000>歌<00:01.500>词\n",
  "translationLyric": "[00:00.000]翻译第一句\n[00:05.240]翻译第二句\n",
  "noLyric": false,
  "provider": "com.netease.cloudmusic",
  "source": "your-player-oplus-lyrics",
  "trackKey": "songId|歌名|歌手|时长毫秒",
  "sessionGeneration": 1
}
```

**字段说明：**

| 字段 | 必填 | 说明 |
|---|---|---|
| `songName` / `artist` / `songId` | 建议 | 歌曲元信息 |
| `lyricType` | ✅ | 固定 `0` |
| `lyric` | ✅ | **逐行歌词**，时间标签 `[mm:ss.mmm]`（毫秒，绝对时间） |
| `rawLyric` | 可选 | 逐字歌词，格式 `[行时间]<字时间>字<字时间>字`（增强 LRC/TTML 才有） |
| `translationLyric` | 可选 | 逐行翻译，时间标签同上 |
| `noLyric` | ✅ | 布尔值，无歌词时 true（此时系统不显示歌词） |
| 其余 | 可选 | 自定义调试信息 |

### 2.3 时间标签格式（关键细节）

- 逐行：`[mm:ss.mmm]` —— 例 `[03:45.120]`
- 逐字：`<mm:ss.mmm>` —— 夹在逐字文本前
- 都是**从歌曲开头的绝对毫秒数**，不是相对上一句的偏移。
- 厂商格式是 `[mm:ss.xx]`（厘秒），但 `[mm:ss.mmm]`（毫秒）系统同样能解析，无需纠结。

---

## 3. 逐条操作步骤

### 3.0 前置准备：拿到播放器源码或可改的 smali

本文案例是「无源码、只有 APK」，所以走 **smali 补丁** 路线。如果你有源码，直接改 Java 更简单
（下面第 3.2 节的 Java 类直接抄进源码工程即可）。

### 3.1 反编译目标 APK

```bash
# 1) 反编译为 Java 源码（用来理解结构、找 MediaSession 管理类、找歌词入口）
jadx -d /tmp/out target.apk --no-res

# 2) 反汇编为 smali（用来做补丁、重打包）
java -cp "baksmali.jar;dexlib2.jar;util.jar;guava.jar;jcommander.jar" \
     org.jf.baksmali.Main d target.apk -o smali_out
#   （或先 unzip 出 classes*.dex，再对单个 dex 执行 baksmali d classes.dex -o out）
```

**要找的关键类：**
- `MediaSession` 管理类（搜索 `new MediaSession`、`setMetadata`、`setPlaybackState`）
- 歌词获取入口（搜索 `GetLyrics`、`Lyrics`、`Lyric`、`TTML`、`LRC`）
- 设置页框架（如果还要加设置项）

### 3.2 编写发布器类（Java，可直接复用）

下面这个类经过真机验证，逻辑完整。替换掉 `AudioPlayerUsers` 等为你播放器自己的歌词获取 API 即可：

```java
package <你的播放器包名>;

import android.content.Context;
import android.media.MediaMetadata;
import android.media.session.PlaybackState;
import org.json.JSONObject;
import java.util.Locale;

public final class OplusLyricPublisher {

    private static final String KEY_LYRIC_INFO = "lyricInfo";
    private static final String KEY_LYRICS_WHOLE = "ucar.media.metadata.LYRICS_WHOLE";
    private static final String KEY_LYRICS_STATUS = "ucar.media.metadata.LYRICS_STATUS";
    private static final String KEY_OPLUS_MEDIA_CONFIG = "oplus.media.config";
    private static final long LYRICS_STATUS_HAS = 101L;
    private static final long LYRICS_STATUS_NONE = 102L;

    /** 在 MediaMetadata.Builder build() 之前调用 */
    public static void attachLyricInfo(MediaMetadata.Builder builder) {
        if (builder == null) return;
        try {
            // 1) 握手键：永远发，无副作用
            builder.putString(KEY_OPLUS_MEDIA_CONFIG, "1");

            // 2) 生成 lyricInfo JSON
            String json = buildLyricInfoJson();
            if (json == null) return;

            builder.putString(KEY_LYRIC_INFO, json);

            // 3) 兼容键
            String lineLrc = new JSONObject(json).optString("lyric", "");
            boolean has = lineLrc != null && !lineLrc.isEmpty();
            builder.putString(KEY_LYRICS_WHOLE, has ? lineLrc : "");
            builder.putLong(KEY_LYRICS_STATUS, has ? LYRICS_STATUS_HAS : LYRICS_STATUS_NONE);
        } catch (Throwable ignore) {
            // 歌词接入失败不得影响原有媒体通知
        }
    }

    private static String buildLyricInfoJson() {
        // TODO: 换成你播放器自己的取歌词 API，返回 String(LRC) 或 TTML 对象
        String title = "歌名";   // <-- 你的播放器取标题
        String artist = "歌手";  // <-- 你的播放器取歌手
        String songId = "id";    // <-- 你的播放器取歌曲 id
        Object lyrics = null;    // <-- 你的播放器取歌词对象（String 或 TTML）

        String lineLrc = null;   // 逐行 LRC
        String wordLrc = null;   // 逐字（可选）
        String transLrc = null;  // 翻译（可选）

        if (lyrics instanceof String) {
            // 普通 LRC / 增强 LRC 文本，直接 parse 成 [mm:ss.mmm]行 格式
            lineLrc = parseLrc((String) lyrics);
        } else if (lyrics instanceof TTML) {
            // ⚠️ 见第 4 节「最大的坑」：逐字 TTML 文本在 spans[].textContent
            lineLrc = buildLineFromTtml((TTML) lyrics);
            wordLrc = buildWordFromTtml((TTML) lyrics);
            transLrc = buildTransFromTtml((TTML) lyrics);
        }

        boolean noLyric = lineLrc == null || lineLrc.isEmpty();
        try {
            JSONObject json = new JSONObject();
            json.put("songName", title == null ? "" : title);
            json.put("artist", artist == null ? "" : artist);
            json.put("songId", songId);
            json.put("lyricType", 0);
            if (!noLyric) json.put("lyric", lineLrc);
            if (wordLrc != null) json.put("rawLyric", wordLrc);
            if (transLrc != null) json.put("translationLyric", transLrc);
            json.put("noLyric", noLyric);
            json.put("provider", "<包名>");
            json.put("source", "your-player-oplus-lyrics");
            return json.toString();
        } catch (Throwable t) {
            return null;
        }
    }

    /** 绝对毫秒 -> [mm:ss.mmm] */
    private static String fmt(int ms) {
        if (ms < 0) ms = 0;
        return String.format(Locale.US, "%02d:%02d.%03d", ms / 60000, (ms / 1000) % 60, ms % 1000);
    }

    // parseLrc / buildLineFromTtml / buildWordFromTtml / buildTransFromTtml 按第 4 节实现
}
```

### 3.3 挂钩子（让发布器被调用）

在播放器 `MediaSession` 管理类里，找到**构建 MediaMetadata 的地方**，在 `build()` 之前插入：

```java
// 原代码类似：builder.build() 或 setMetadata(builder.build())
OplusLyricPublisher.attachLyricInfo(builder);
mediaSession.setMetadata(builder.build());
```

钩子触发点建议（有则挂）：
- 切歌 / 首次加载歌词时（`UpdateMediaMetadata` / `UpdateLyrics`）
- 逐行歌词刷新标题时（避免重复解析，可在发布器内做 trackKey 缓存）

### 3.4 改包名（若播放器不在白名单）

用 aapt2 反编译 manifest，改 `package` 属性为白名单内包名（如 `com.netease.cloudmusic`）：
> 注意：若播放器组件用**绝对路径**类名（`music.hifistatus.Xxx`），改 package 不影响组件引用；
> 若用相对路径（`.Xxx`），需同步改所有组件名。

### 3.5 重打包 + 签名（smali 路线）

```bash
# 1) smali 重新汇编为 dex
java -cp "smali.jar;baksmali.jar;dexlib2.jar;util.jar;guava.jar;jcommander.jar;antlr.jar;antlr4.jar" \
     org.jf.smali.Main a -o classes.dex smali_out

# 2) 替换原 APK 里的 classes.dex（用 Python zipfile）
#    repack.py：读原 APK 所有条目，替换 classes.dex，写新 APK

# 3) 对齐
zipalign -p -f 4 out.apk out_aligned.apk

# 4) 签名（v1+v2+v3 都要，否则新版 Android 拒装）
apksigner sign --ks keystore --ks-key-alias alias --ks-pass pass:xxx \
    --key-pass pass:xxx --v1-signing-enabled true --v2-signing-enabled true \
    --v3-signing-enabled true out_aligned.apk
```

### 3.6 安装验证

```bash
adb install -r out_aligned.apk
# 播放一首**有歌词**的歌，锁屏 / AOD 查看锁屏岛
```

---

## 4. 踩坑经验（务必读完，每条都是真金白银换来的）

### ⭐ 坑 1：逐字 TTML 的文本在 `spans[].textContent`，不在 `lyricsTextContent`（本次最核心的坑）

很多播放器（含 Melody）的歌词对象是 `TTML` 结构。**有逐字 span 的段落，`lyricsTextContent` 字段是空的**，
真正的文本分散在 `paragraph.spans[].textContent`。只读 `lyricsTextContent` → 所有段落被跳过 →
发布 `noLyric:true` → 锁屏岛永远没歌词。

**正确写法（段落取文本）：**

```java
String text = paragraph.lyricsTextContent;
if (text == null || text.trim().isEmpty()) {
    if (paragraph.spans != null) {
        StringBuilder sb = new StringBuilder();
        for (Span span : paragraph.spans) {
            if (span != null && span.textContent != null) sb.append(span.textContent);
        }
        text = sb.toString();
    }
}
```

**快速自查方法**：在发布器里加一行 `Log.i`，打印 `noLyric=` 和 `lineLrc.length()`。
如果播放器明明有歌词却输出 `noLyric=true / 长度 0`，基本就是这个问题。

### 坑 2：新版系统调不出歌词，先怀疑 `oplus.media.config` 握手

`resolveLyricSupport` 在 RUS `lyricEnable==1` 时强制要求 `oplus.media.config=="1"`。
这是云端下发的，版本会变。**永远无条件发布握手键**，一劳永逸。

### 坑 3：网易云等 app 本身不原生发 `lyricInfo`

手机版网易云只发 `ucar.media.metadata.LYRICS_WHOLE`，且受 `UCarService`（车机）门控；
它的锁屏歌词实际靠 root Provider 补发。**别拿网易云当"应该发什么"的参照**，
参照厂商亲生应用「欢太音乐 com.heytap.music」的完整键位（lyricInfo JSON + 全套 ucar 键）。

### 坑 4：smali 插桩时注意寄存器上限

在大型方法（如页面管理方法，19 寄存器 3 参数）里插桩，新增 `move-result-object v16`
会触发「寄存器越界」。**优先挑小方法挂钩子**，或先 `baksmali d` 确认方法头部 `.registers N`，
只使用 `< N` 的寄存器编号；搞不定就退化为「总开关」这类简单方案，别死磕。

### 坑 5：探针读 metadata 时的两个坑

写 MediaController 探针应用（见 5.2）时：
1. `getActiveSessions(null)` 会 `SecurityException` —— **必须传自己的 ComponentName**：
   `getActiveSessions(new ComponentName(ctx, 自己的NotificationListenerService.class))`
2. ColorOS 限制无启动 Activity 的应用绑定通知监听服务 —— **探针必须带一个 LAUNCHER Activity**。
3. 对应用 `force-stop` 后它会进入「停止状态」，系统拒绝绑定监听服务 —— 换实现要**卸载重装**。

### 坑 6：别信 `dumpsys media_session` 的音频 usage 字段

`dumpsys media_session` 显示的 `usage=USAGE_ASSISTANCE_SONIFICATION` 可能是会话未播放时的
残留值，实际播放时是 `USAGE_MEDIA`。别被它误导去改音频属性。

---

## 5. 调试方法（区分「发布问题」还是「门控问题」）

### 5.1 看系统日志判断门控是否放行

```bash
adb logcat -c
adb logcat -v time | grep --line-buffered "SystemUi--Media" > live.log
# 播放你的 app 的歌曲，然后：
grep "你的包名" live.log | grep -oE "lyricSupport=[a-z]+" | sort | uniq -c
```

- 出现 `lyricSupport=true` 多条 → **门控全通，问题在歌词内容**（去查坑 1）。
- 一条都没有 → **门控没放行**（查包名白名单 / 握手键 / 黑名单）。

### 5.2 写 MediaController 探针读真实 metadata（最强手段）

写一个 `NotificationListenerService`，在 `onListenerConnected` 里遍历所有活跃会话，打印每个
metadata 键的值（`lyricInfo`、`ucar.media.metadata.*`、`oplus.media.config` 等）。

这能直接看到：**你的 app 到底把歌词发布成什么样了**。对比官方 app（欢太音乐）的键，缺什么补什么。

关键代码骨架：

```java
MediaSessionManager msm = (MediaSessionManager) getSystemService(MEDIA_SESSION_SERVICE);
List<MediaController> cs = msm.getActiveSessions(
    new ComponentName(this, ProbeService.class));   // ← 必须传自己的 ComponentName
for (MediaController c : cs) {
    String pkg = c.getPackageName();
    MediaMetadata md = c.getMetadata();
    if (md == null) continue;
    Log.i("Probe", "pkg=" + pkg + " lyricInfo=" + md.getString("lyricInfo"));
    // ... 打印所有你关心的键
}
```

### 5.3 对照组：官方 app 的 metadata 全键

```bash
adb shell dumpsys media_session | sed -n '/package=com.heytap.music/,/^      package=/p'
```

记录官方 app 发布的所有键和值，作为你的发布目标的**黄金对照**。

---

## 6. 工具清单（按用途分类）

### 6.1 反编译 / 反汇编

| 工具 | 用途 | 获取 |
|---|---|---|
| **jadx** | APK → Java 源码（理解结构、定位类） | GitHub: skylot/jadx |
| **baksmali** | dex → smali（做补丁） | Maven Central `com.android.tools.smali:smali-baksmali:2.5.2` |
| **smali** | smali → dex（重汇编） | Maven Central `com.android.tools.smali:smali:2.5.2` |

> baksmali/smali 运行时依赖：`dexlib2`、`util`、`guava`、`jcommander`、`antlr`、`antlr4`。

### 6.2 Android SDK 构建工具（build-tools，如 36.0.0）

| 工具 | 用途 |
|---|---|
| **aapt2** | dump 清单/权限（`dump xmltree`/`dump badging`）、link 打包、查启动组件 |
| **d8** | Java `.class` → `.dex` |
| **zipalign** | APK 对齐（`-p -f 4`） |
| **apksigner** | 签名（`sign` / `verify`），v1+v2+v3 |

### 6.3 设备调试

| 工具 | 用途 |
|---|---|
| **adb** | 安装、shell、`dumpsys media_session`、`logcat`、`pm list packages`、`input keyevent`（模拟媒体键） |

常用命令速查：

```bash
adb devices -l                                   # 确认连接
adb shell pm list packages | grep -i cloudmusic   # 查已装包
adb shell dumpsys media_session                   # 看活跃媒体会话与 metadata
adb shell dumpsys package <pkg>                   # 查版本号/安装时间
adb shell input keyevent 87                       # 媒体键「下一曲」（触发 metadata 刷新）
adb shell cmd notification allow_listener <组件>   # 启用通知监听（探针用）
```

### 6.4 编译与脚本

| 工具 | 用途 |
|---|---|
| **JDK**（javac `--release 17`） | 编译 Java 补丁类 / 探针 |
| **Android platform `android.jar`**（如 android-34） | javac/d8 的编译 classpath |
| **Python 3**（`zipfile`） | APK 重组、manifest 二进制补丁、探针打包 |

---

## 7. 完整工作流速览（TL;DR）

```text
1. jadx 反编译 → 定位 MediaSession 管理类 + 歌词获取 API
2. 写 OplusLyricPublisher（第 3.2 节），挂到 setMetadata 之前
3. 注意 TTML 文本在 spans[].textContent（坑 1）
4. 无条件发 oplus.media.config="1"（坑 2）
5. 改包名为白名单内（com.netease.cloudmusic）
6. smali/d8 汇编 → zipalign → apksigner v1+v2+v3
7. adb 装真机 → 播放有歌词的歌 → logcat 查 lyricSupport
8. 不显示？用 MediaController 探针读 metadata，对照官方 app 补键
```

---

## 8. 附：可复用的经验原则

1. **先上真机读数据，再猜原因**。逆向的结论都要用真机 `dumpsys`/`logcat`/探针验证，
   纯静态分析极易误判（本次一度误判为"系统门控问题"，实际是 TTML 解析 bug）。
2. **永远拿厂商亲生应用当参照**（欢太音乐），别拿需要 root Provider 兜底的第三方（网易云）当参照。
3. **握手键、兼容键多发无害**：系统不认识的键会被忽略，多发布几个键不会报错。
4. **smali 插桩挑小方法**，大方法先确认寄存器上限。
5. **探针/调试 APK 要带 LAUNCHER Activity**，且别对调试 app 用 `force-stop`。

---

*本指南源自一次完整的真实适配（Melody-Domain 播放器 → ColorOS 16 锁屏岛歌词，真机验证通过），
含机制分析、数据契约、逐条步骤、工具清单与踩坑经验，可直接作为其他 AI 复现同类适配的操作手册。*
