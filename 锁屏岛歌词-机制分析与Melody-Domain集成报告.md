# ColorOS 16+ 音乐锁屏岛歌词 —— 机制分析与 Melody-Domain 集成报告

> 分析对象：`网易云音乐-9.6.10(9006010).apk`（**com.netease.cloudmusic 官方版**）、
> `音乐.apk`（**com.heytap.music 欢太音乐**，ColorOS 自带官方音乐）、
> `音乐-40.1.17.6.0`（欢太音乐最新版）、`ColorOS-Live-Lyrics-Bridge-main`（v4.5.2）、
> `Melody-Domain_v1.0.9.9999.APK`（music.hifistatus 旋律之域）
> 分析工具：jadx 1.5.6、baksmali/smali 2.5.2、aapt2、apksigner、REA MCP、真机 MediaController 探针
> 日期：2026-10-25（v3 终版，经真机 ColorOS 16 联调验证）

## ⭐ 最终结论（真机验证后）

**锁屏岛歌词已成功实现并验证通过。** 完整的因果链：

1. **数据通道**：播放器自身 `MediaSession` 的 `MediaMetadata["lyricInfo"]`（JSON，含逐行 `lyric`、
   可选逐字 `rawLyric`、翻译 `translationLyric`）+ 进度取 `PlaybackState` 时钟。**无需 root**。
2. **包名验证**：SystemUI 媒体管线按包名放行（RUS 白名单 + 歌词入口 + lyricSupport 门控），
   改包名为白名单内的 `com.netease.cloudmusic` 即可通过；**不校验签名**。
3. **真根因（本次修复的关键）**：Melody 的歌词是 `Lyrics.TTML` 对象，逐字 TTML 的段落文本存在
   `paragraph.spans[].textContent`（有 span 时 `lyricsTextContent` 为空）。初版解析只读
   `lyricsTextContent` → 全部段落被跳过 → 发布 `noLyric:true` → 锁屏岛无歌词。
   **修复**：`lyricsTextContent` 为空时从 spans 拼接文本。
4. 真机验证：修复后 `noLyric=false`、行歌词 + 逐字 + 翻译全部正确发布，锁屏岛歌词显示成功。

---

## 0. 先回答：需要 Root 吗？

**不需要。** 分层说明：

| 层 | 归属 | 需要 Root？ |
|---|---|---|
| 歌词数据发布（`MediaMetadata["lyricInfo"]` + `ucar.media.metadata.*`） | 播放器自身 MediaSession，标准 Android API | **否** |
| 锁屏岛/锁屏歌词页显示 | ColorOS SystemUI 原生消费（厂商系统自带） | **否** |
| Bridge 增强（逐字渲染、翻译按钮、亮度/字号/风格、AOD、字符上浮动画） | Xposed 模块，作用域 `system` + `com.android.systemui` | **是（LSPosed）** |

依据 Bridge 协议原文：「播放器不需要依赖 Bridge APK、不发送 Bridge 私有广播，也不进入 Bridge 的
LSPosed 作用域……**未安装 Bridge 时 ColorOS 仍可原生消费**；Bridge 只是 SystemUI 侧的可选增强。」
本集成的 Melody-Domain 走的就是纯数据发布路线，装完即用；想要 Bridge 的高级视觉效果才需 root。

### 0.1 ColorOS 系统侧准入链全图（Bridge 逆向证据 + 双厂商实测）

```text
播放器 MediaSession.setMetadata(...)
        │
        ▼
system_server: com.android.server.media.OplusMediaControlService
   ├─ isInHistoryPlayInfoWhiteList(pkg)      ← 媒体历史白名单（包名）
   ├─ isInMediaBlackList(pkg)                ← 媒体黑名单（包名；被拉黑则整条链路断）
        │
        ▼
SystemUI: OplusMediaDataManagerExImpl.onMetaDataChanged(pkg, metadata)
   ├─ MediaActionPrioritySelectorImpl.getRusWhiteList()        ← RUS 云端白名单（包名 List）
   ├─ getLyricEntrance(pkg):int                                ← 歌词入口等级
   ├─ resolveLyricSupport(pkg, metadata):boolean               ← RUS lyricEnable 门控：
   │      getLyricEnable(pkg) != 1  → true（放行）
   │      getLyricEnable(pkg) == 1  → metadata.getString("oplus.media.config")=="1"（要握手！）
   ├─ loadLyricInBg(pkg, metadata) → metadata.getString("lyricInfo")  ← 读歌词 JSON
        │
        ▼
MediaLiveAlertController: isLyricSupported = lyricSupport && (displayPolicy & 16) != 0
        │
        ▼
锁屏岛 / 锁屏歌词页渲染（进度取 PlaybackState 时钟）
```

**证据文件**：Bridge `app/src/test/resources/fixtures/systemui-lyric-support-gate-evidence.txt`
（SystemUI 16.99.12 / 17.99.02 双版本 bytecode 级取证）。

**v2 修复关键**：`oplus.media.config=1` 握手。RUS 是云端下发配置，最新 ColorOS 16 可能已对
`com.netease.cloudmusic` 等包置 `lyricEnable=1`——此时**必须**播发该握手，否则
`resolveLyricSupport` 返回 false，歌词整条链路断掉。这正是「旧版厂商应用 / 初版集成在最新
系统调不出锁屏岛歌词」的最可能原因。`lyricEnable!=1` 时系统忽略此键，播发无副作用。

### 网易云 9.6.10 的原生发布真相（重要修正）

网易云**不原生发布 `lyricInfo` JSON 键**，只在 `music_biz_hicar_release` 模块发布
`ucar.media.metadata.LYRICS_WHOLE`/`LYRICS_STATUS`，且受 `UCarService` 连接门控
（`q90/d.java`）。即：**网易云在纯手机场景的锁屏歌词依赖 player-netease Provider（root）补发**；
欢太音乐（com.heytap.music，厂商亲生应用）才原生发布完整 `lyricInfo` JSON + 全部 ucar 键。
本集成按欢太的完整键位发布（lyricInfo JSON + LYRICS_WHOLE + LYRICS_STATUS + v2 新增的
`oplus.media.config` 握手），不依赖任何 root 组件。

---

## 1. 锁屏岛歌词的通信机制

### 1.1 两个官方应用的实测结论

**欢太音乐（com.heytap.music，厂商亲生应用）** —— classes6.dex 内嵌 Flamingo 引擎：
`com.allsaints.music.player.thirdpart.MediaSessionManager#buildCommonMetedata` 发布：

```kotlin
// 主协议键
builder.putString("lyricInfo", json)          // JSON: songName/artist/songId/lyricType=0/id/lyric/noLyric
// OPLus 媒体管线兼容键
builder.putString("ucar.media.metadata.LYRICS_WHOLE", 整篇逐行LRC)
builder.putLong("ucar.media.metadata.LYRICS_STATUS", 101L/102L)   // 101=有歌词 102=无
builder.putString("ucar.media.metadata.LYRICS_LINE", 当前行)
builder.putString("ucar.media.metadata.UCAR_TITLE"/"UCAR_ARTIST"/"COLLECT_STATE"/"PLAY_MODE"...)
```

**网易云音乐（com.netease.cloudmusic 9.6.10）** —— 模块 `music_biz_hicar_release`：
- 歌词数据模型（`meta/ya.java`）：`lrc`（逐行）/`klyric`（卡拉OK逐字）/`tlyric`（逐行翻译）/
  `romalrc`（罗马音）/`yrc`（网易云逐字格式）/`ytlrc`/`yromalrc`，各带 version；
- 发布链路：`module.lyric.f` 歌词回调 → `r90.b`（HiCar MediaSession 委托）→
  消息(5, lyric) → `q90.d.a(status, lyric, metadata, session)`：

```java
// q90/d.java —— 受 UCarService 连接状态门控
if (status == 0) {
    builder = new MediaMetadataCompat.Builder(t90.a.b(mediaMetadata)); // 补全部 ucar.* 键
    if (lyric 为空)      putLong("ucar.media.metadata.LYRICS_STATUS", 1)
    else if (超12万字符)  putLong("ucar.media.metadata.LYRICS_STATUS", 1)   // 超限丢弃
    else putLong("ucar.media.metadata.LYRICS_STATUS", 0)
            .putString("ucar.media.metadata.LYRICS_WHOLE", lyric);
    mediaSession.setMetadata(...);
}
// status: 2=开始加载 3=错误 4=歌词文本
```

**关键差异**：网易云自己只往 MediaSession 发布**逐行 LRC**（`LYRICS_WHOLE`），
逐字/翻译依赖 Bridge 生态的 `player-netease` Provider（Xposed）按运行 profile 追加
——这正是需要 root 的增强层。欢太音乐则连 `lyricInfo` JSON 一起发。

### 1.2 锁屏岛歌词的"另外的 APK"在哪

用户猜测正确——**锁屏岛的显示端在 ColorOS SystemUI（系统 APK，本地未导出）**：
- 消费播放器 MediaSession 的 `lyricInfo` / `ucar.media.metadata.LYRICS_WHOLE`；
- 内部通过 `LyricsRecyclerView` 等私有视图渲染（Bridge 仓库的 `systemui/lyrics/` 目录
  就是对这套私有视图的 Hook 适配，可作为系统侧结构的参考）；
- 播放器数据端 + Bridge 适配端 = 完整链路的两半，都在本地（两个音乐 APK + Bridge 源码）。

### 1.3 数据契约汇总（两厂商 + Bridge 协议三方一致）

| 键 | 欢太 | 网易云 | Bridge 协议 | 用途 |
|---|---|---|---|---|
| `MediaMetadata["lyricInfo"]` JSON | ✓ | —（由 Provider 补） | 主协议 | 锁屏歌词页主数据 |
| `ucar.media.metadata.LYRICS_WHOLE` | ✓ | ✓ | OPLus 兼容 | 整篇逐行 LRC |
| `ucar.media.metadata.LYRICS_STATUS` | ✓(101/102) | ✓(0/1) | OPLus 兼容 | 歌词状态 |
| `rawLyric`（逐字）/`translationLyric` | 该版本未发 | 未发（Provider 追加） | 可选扩展 | 卡拉OK/翻译 |
| `PlaybackState` 时钟 | ✓ | ✓ | 必需 | 进度不靠重写 metadata |
| `TOGGLE_TRANSLATION` CustomAction | — | — | 可选 | 翻译按钮 |

**包名验证（结论不变）**：SystemUI 歌词入口按包名放行（Bridge `PlayerSystemUiPolicy` 22 个白名单，
网易云 `com.netease.cloudmusic` 在内）；数据协议本身开放，**不校验签名**
（网易云签名 `CN=LiangJian,OU=Corp.Netease,O=CloudMusic`，欢太签名 `CN=HeyTapTeam`，
两者互不相同，系统若校验签名则厂商自家应用都无法跨设备通用——实测口径是包名）。

---

## 2. Melody-Domain 集成实现

### 2.1 架构分析结论

- 包名 `music.hifistatus`（旋律之域），原生 Java，minSdk 16 / targetSdk 35；
- 播放核心：`music.hifistatus.MediaSessionManager`（MediaSession 唯一管理类）；
- 歌词引擎：`Lyrics.LyricsParser`（LRC/增强 LRC 逐字，毫秒）、`Lyrics.TTML`（逐字+翻译）；
- 歌词入口：`AudioPlayerUsers._AudioPlayer.GetLyrics()` → `String`(LRC) 或 `TTML`。

### 2.2 集成方案（最小侵入）

新增 1 个类 + 3 个钩子（全部在 `MediaSessionManager.smali`，共 6 行）：

**`music.hifistatus.OplusLyricPublisher`**：
- `attachLyricInfo(MediaMetadata.Builder)`：取当前歌词 → 生成
  - `lyricInfo` JSON（songName/artist/songId/lyricType=0/lyric/noLyric，
    可选 rawLyric 逐字、translationLyric 翻译、trackKey/sessionGeneration）
  - `ucar.media.metadata.LYRICS_WHOLE` + `LYRICS_STATUS`(101/102)（对齐两厂商键位）
  - 按 trackKey 缓存；逐字仅增强 LRC/TTML 才生成（不伪造扫光）
- `ensureTranslationAction(PlaybackState.Builder)`：有翻译时补发翻译按钮（防重复累积）。

**钩子**：`UpdateMediaMetadata()` / `_UpdateLyrics()` 后 attachLyricInfo；
`_SetPlaybackState(ZJ)` build 前 ensureTranslationAction。

### 2.3 包名与版本（对齐网易云）

- `package`：`music.hifistatus` → **`com.netease.cloudmusic`**（仅改 package 属性；
  Melody 组件名全部是绝对路径 `music.hifistatus.Xxx`，不受影响）
- **`versionCode`：20 → `9006010`；`versionName`：`1.0.9.9999` → `9.6.10`**（与网易云 9.6.10 完全一致）
- 插入 `<meta-data android:name="io.github.andrealtb.lockscreenlyrics.OPLUS_MEDIA_HISTORY" android:value="true"/>`
- baksmali → 补丁 → smali 重汇编 → 重组 → zipalign → apksigner v1+v2+v3 重签名
  （自签证书）。resources.arsc 与 classes2.dex 未动。

---

## 3. 产物与安装

| 文件 | 说明 |
|---|---|
| `Melody-Domain_9.6.10(9006010)_锁屏岛歌词.apk` | **最终版**：com.netease.cloudmusic / 9006010 / 9.6.10，TTML 解析已修复，锁屏岛歌词验证成功 |
| `.cache/` | 中间版本（v1/v2/v3debug/1.1.0）已归档，可自行清理 |
| 本报告 | 机制分析 + 实现说明 + 真机调试过程 |

安装：**先卸载官方网易云音乐**（包名冲突）→ 安装本 APK → 播放本地音乐 → 锁屏/AOD 查看。
可选增强（需 root）：安装同 Release 的 Bridge APK，LSPosed 作用域勾选 `system` +
`com.android.systemui`，重启手机。

已知注意点：
- 逐字/翻译依赖歌词文件本身（增强 LRC 或 TTML）；网易云自身也只发逐行，逐字靠 Bridge；
- Melody 设置里"歌词写入标题"开启时标题会变当前歌词行，Bridge 能识别，异常时可关闭；
- 系统大版本更新可能改变 SystemUI 结构，需重新验证。

## 4. 复现环境

- jadx 1.5.6；baksmali/smali 2.5.2（Maven Central，主类 org.jf.*.Main，
  需 dexlib2/util/guava/jcommander/antlr 依赖）；
- Android SDK build-tools 36.0.0（aapt2/d8/zipalign/apksigner）+ JDK 21；
- Python 脚本 `patch_manifest.py`（二进制 AXML 补丁）/`repack.py`（APK 重组）；
- 构建工作区 `<临时构建目录>/build/`（smali 树、Java 源码、脚本齐全）。
