# FFXIV 2.0 WMV 影片播放問題分析

## 問題概述

在 XIV-on-Mac 上執行 FFXIV 時，2.0 版本（A Realm Reborn）的過場動畫無法正常播放，出現黑屏或無聲音的情況。

## 影片檔案分析

根據 MediaInfo 分析，FFXIV 2.0 的過場動畫檔案（位於 `game/movie/ffxiv/`）使用以下格式：

| 檔案 | 大小 | 時長 | 視訊編碼 | 音訊編碼 | 建立日期 |
|------|------|------|----------|----------|----------|
| 00000.dat | 202 MiB | 3:37 | VC-1 (WMV3) | 日語音軌 | 2013-05-03 |
| 00001.dat | 498 MiB | 8:18 | VC-1 (WMV3) | WMA Pro 6聲道 | 2013-06-05 |
| 00002.dat | 96 MiB | 1:40 | VC-1 (WMV3) | 日語音軌 | 2013-05-07 |

### 技術規格

- **容器格式**: Windows Media (ASF/WMV)
- **視訊編碼**: VC-1 (WMV3) MP@HL
- **解析度**: 1280×720 (16:9)
- **幀率**: 29.970 FPS
- **音訊編碼**: WMA (Windows Media Audio) Pro
- **音訊規格**: 44.1 kHz, 16 bits, 6聲道

## 問題根本原因

### 1. Microsoft 專有編碼器

VC-1 和 WMA 是 Microsoft 的專有編碼格式：

- **VC-1**: 基於 Windows Media Video 9，是 Microsoft 為 Blu-ray 和串流開發的視訊編碼
- **WMA Pro**: Microsoft 的專有音訊編碼，支援多聲道

這些編碼器在 Windows 上由 **Media Foundation** API 提供解碼支援。

### 2. Wine 的 Media Foundation 限制

```
FFXIV.exe → Media Foundation API → ??? → 影片播放失敗
```

Wine 對 Media Foundation 的實作有以下限制：

| 組件 | Windows | Wine (macOS) |
|------|---------|--------------|
| Media Foundation API | ✅ 完整支援 | ⚠️ 部分實作 |
| VC-1 解碼器 | ✅ 內建 | ❌ 缺失 |
| WMA Pro 解碼器 | ✅ 內建 | ❌ 缺失 |
| DirectShow 後備 | ✅ 可用 | ⚠️ 有限 |

### 3. XIV-on-Mac Wine 配置

XIV-on-Mac 使用自訂編譯的 Wine 10.0，配置如下：

```nix
# wine-builder/default.nix
configureFlags = [
  "--without-gstreamer"  # 禁用 GStreamer 後端
  # ... 其他選項
];
```

由於 GStreamer 在 macOS 上的相容性問題，XIV-on-Mac 禁用了此後端，導致傳統的 Wine Media Foundation 解決方案無法運作。

## 為何 3.0+ 影片正常播放？

FFXIV 從 3.0 (Heavensward) 開始改用 **Bink Video 2** (.bk2) 格式：

| 版本 | 影片格式 | 編碼 | Wine 支援 |
|------|----------|------|-----------|
| 2.0 ARR | WMV | VC-1 + WMA | ❌ 需要 Media Foundation |
| 3.0+ HW/SB/ShB/EW/DT | Bink 2 | Bink Video | ✅ 遊戲內建解碼器 |

Bink Video 使用遊戲內建的解碼器，不依賴系統 API，因此在 Wine 下正常運作。

## 解決方案

### 已實作：FFmpeg Media Foundation 後端

Wine 10.0 引入了 `winedmo.dll`，提供基於 FFmpeg 的 Media Foundation 後端：

```
FFXIV.exe → Media Foundation API → winedmo.dll → FFmpeg → 影片播放
```

#### 啟用方式

1. 將 FFmpeg 函式庫打包進 Wine
2. 設定 Registry 啟用 FFmpeg 後端：
   ```
   HKCU\Software\Wine\MediaFoundation
   DisableGstByteStreamHandler = 1
   ```
3. 設定 `DYLD_LIBRARY_PATH` 指向 FFmpeg 函式庫

#### 支援的編碼

FFmpeg 支援 VC-1 和 WMA 解碼，理論上可以播放 FFXIV 2.0 的 WMV 影片。

## 相關資源

- [Wine 10.0 Release Notes](https://www.winehq.org/announce/10.0)
- [Wine Bug #49692 - Media Foundation WMV Support](https://bugs.winehq.org/show_bug.cgi?id=49692)
- [FFmpeg VC-1 Decoder](https://ffmpeg.org/ffmpeg-codecs.html#vc1)

## 附錄：影響的過場動畫

| 檔案 | 內容描述 |
|------|----------|
| 00000.dat | 遊戲開頭 CG（選擇城市後播放）|
| 00001.dat | 2.0 片頭動畫「Answers」|
| 00002.dat | 飛艇過場動畫 |

這些影片僅在 2.0 主線劇情中出現，影響新玩家的遊戲體驗。進入 3.0 後的所有過場動畫均可正常播放。
