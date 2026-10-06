# [FIX] Android 15 Compatibility Patches for Legacy Kernels (v4.4)

**Device:** Sony Xperia XZ1 Compact (yoshino / MSM8998)
**OS:** Android 15 (LineageOS)
**Kernel:** 4.4.x

本文件記錄了在將 Android 15 移植至運行舊版 Kernel (4.4) 的 MSM8998 裝置時，所遇到的兩個致命系統相容性問題及其 Kernel 層級的修復方法。這避免了修改 AOSP 原始碼，確保了 ROM 的純淨度。

---

## Issue 1: LMKD Crash (Failed to get free memory)

### 🚨 現象 (Symptoms)
系統頻繁在背景觸發重啟，Logcat 持續報錯：
```text
lowmemorykiller: /proc/zoneinfo parse error
lowmemorykiller: Failed to get free memory!
```
導致 Android 15 的 `lmkd` 完全失效，無法在記憶體壓力下正確回收資源，最終引發 OOM (Out Of Memory) 整機當機。

### 🔍 根因 (Root Cause)
Android 15 的 `lmkd` 對 `/proc/zoneinfo` 的解析格式與單位有著極為嚴格的要求，而舊版 Kernel 存在兩項不相容：
1. **缺少標記：** 新版 `lmkd` 強制要求每個 NUMA Node 必須包含 `  per-node stats` 標記，否則會判定解析失敗並中斷。Kernel 4.4 未輸出此標記。
2. **單位誤導 (Unit Mismatch)：** Kernel 內部的計數器 `NR_INDIRECTLY_RECLAIMABLE_BYTES` 是以 Bytes 測量，但輸出給 User-space 的名稱卻是 `nr_indirectly_reclaimable`。這導致 `lmkd` 誤將 Bytes 數值當作 Pages 計算，產生出荒謬的可用記憶體數值（例如 3.4MB 被算成 13.8GB）。

### 🛠️ 解決方案 (The Patch)
修改 Kernel 中負責輸出 zoneinfo 的邏輯，強行插入相容性標記，並在輸出 `nr_indirectly_reclaimable` 前，將 Bytes 轉換為 Pages。

**File: `mm/vmstat.c`**
```c
 static void zoneinfo_show_print(struct seq_file *m, pg_data_t *pgdat,
                                                         struct zone *zone)
 {
        int i;
        seq_printf(m, "Node %d, zone %8s", pgdat->node_id, zone->name);

+       /* [FIX] Android 15 lmkd compatibility: Insert required node marker */
+       if (zone == pgdat->node_zones) {
+               seq_printf(m, "\n  per-node stats"
+                       "\n      nr_inactive_file %lu"
+                       "\n      nr_active_file   %lu",
+                       node_page_state(pgdat, NR_INACTIVE_FILE),
+                       node_page_state(pgdat, NR_ACTIVE_FILE));
+       }

        // ... (保持原有的 pages free 等輸出) ...

-       for (i = 0; i < NR_VM_ZONE_STAT_ITEMS; i++)
-               seq_printf(m, "\n    %-12s %lu", vmstat_text[i],
-                               zone_page_state(zone, i));
+       /* [FIX] Android 15 lmkd unit mismatch: Convert Bytes to Pages */
+       for (i = 0; i < NR_VM_ZONE_STAT_ITEMS; i++) {
+               unsigned long val = zone_page_state(zone, i);
+               if (i == NR_INDIRECTLY_RECLAIMABLE_BYTES) {
+                       val >>= PAGE_SHIFT; /* Bytes to Pages */
+               }
+               seq_printf(m, "\n    %-12s %lu", vmstat_text[i], val);
+       }
 }
```

---

## Issue 2: Graphical Glitches (Tearing / Artifacts / Black Screen)

### 🚨 現象 (Symptoms)
在 Pixel Launcher 等高度依賴即時模糊與複雜圖層混合的場景中，螢幕會偶發性出現嚴重的花屏 (Artifacts) 或畫面撕裂。
Logcat 中充斥著大量來自 `SurfaceFlinger` 與 `Chromium` 的時序錯誤：
```text
E SurfaceFlinger: Out of order buffers detected for RequestedLayerState...
E chromium: [ERROR:ui/gfx/gpu_fence.cc:104] No timestamp provided from sync_(pt|fence)_info for fd...
```

### 🔍 根因 (Root Cause)
Android 15 的圖形堆疊（特別是 RenderEngine 與 `libsync`）在處理 `sync_fence`（同步柵欄）時，嚴格要求取得精確的時間戳記 (`timestamp_ns`) 以排序圖層渲染。
在舊版 Kernel (例如未完全主線化 dma-buf 的 4.4) 的 `drivers/base/sync.c` 實作中，當 IOCTL 請求 `sync_pt_info` 時，如果底層 GPU 驅動沒有主動更新信號時間，Kernel 就會回傳 `timestamp_ns = 0`。這會導致 SurfaceFlinger 失去時序基準，進而將新舊畫面的 Buffer 排錯順序（Out of order buffers），直接引發花屏。

### 🛠️ 解決方案 (The Patch)
在 Kernel 的 `sync.c` 中加入 Workaround。當 Fence 的狀態為「已發出信號 (signaled)」但時間戳卻為 0 時，強行塞入當前系統時間 (`ktime_get()`)，以滿足 Android 15 對時序單調遞增的嚴格檢查。

**File: `drivers/base/sync.c` (或對應的 `sync_file.c`)**
```c
 static int sync_fill_pt_info(struct fence *fence, void *data, int size)
 {
        // ... (保持原有的初始化代碼) ...

        strlcpy(info->obj_name, fence->ops->get_timeline_name(fence),
                sizeof(info->obj_name));
        strlcpy(info->driver_name, fence->ops->get_driver_name(fence),
                sizeof(info->driver_name));

-       if (fence_is_signaled(fence))
-               info->status = fence->status >= 0 ? 1 : fence->status;
-       else
-               info->status = 0;
-       info->timestamp_ns = ktime_to_ns(fence->timestamp);

+       /* [FIX] SurfaceFlinger out-of-order buffers: Provide a valid timestamp */
+       if (fence_is_signaled(fence)) {
+               info->status = fence->status >= 0 ? 1 : fence->status;
+               
+               /* If driver didn't provide a timestamp, fallback to current time */
+               if (ktime_to_ns(fence->timestamp) == 0) {
+                       info->timestamp_ns = ktime_to_ns(ktime_get());
+               } else {
+                       info->timestamp_ns = ktime_to_ns(fence->timestamp);
+               }
+       } else {
+               info->status = 0;
+               info->timestamp_ns = 0; /* Unsignaled fences must return 0 */
+       }

        return info->len;
 }
```

---
**Maintainer Notes:**
* These patches eliminate the need for detrimental properties like `debug.sf.latch_unsignaled=1`, which inherently causes tearing.
* By fixing the root cause in the kernel, the AOSP/LineageOS source tree remains untouched and conflict-free for future repo syncs.