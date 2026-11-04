# AI Upscale 繁體中文離線版

> 單一 HTML 檔的 AI 圖片放大工具。**Real-ESRGAN 模型直接內嵌在檔案裡**，
> 圖片與推論全部在你的電腦上完成，離線也能用。

---

## 一、檔案

| 檔案 | 大小 | 說明 |
|------|------|------|
| `output/AI-Upscale-繁體中文離線版.html` | 6584749 bytes（6.28 MB） | 交付檔，單一檔案、免安裝 |

**用法**：雙擊用瀏覽器（Chrome / Edge 113+ / Safari 26+）開啟即可。
不需要安裝、不需要伺服器、**不需要網路**。

> 需要 **WebGPU** —— 這是神經網路推論，不是像素複製。
> 純 CPU 跑這個模型，一塊 128×128 就要 7 秒以上，一張照片得跑十幾分鐘；
> 交給 GPU 则是秒級。沒有 WebGPU 時頁面會直接說明原因。

---

## 二、核心驗證：原始工具會不會上傳圖片？

這個專案的做法是「先量測、再相信」。我用真實 Chromium（Playwright）實際開啟
EZOnline 的 AI 圖片放大頁面、拖入圖片、觸發放大，並記下每一個網路請求。

### 2.1 原始工具（EZOnline 線上版）的實測結果

```
=== 上傳圖片期間新增的請求 ===
  GET blob:https://ezonline.tools/7a065db2-...      （本機預覽）

=== 執行放大期間新增的請求 ===
  GET  ezonline.tools/_astro/index.B_sbyPE-.js
  GET  ezonline.tools/_astro/runtime.DEbNJulq.js
  HEAD huggingface.co/Heliosoph/realesrgan-onnx/.../realesr-general-x4v3.onnx
  HEAD ezonline.tools/ai/upscale/09b757ac.../realesr-general-x4v3.onnx
  HEAD us.aws.cdn.hf.co/xet-bridge-us/...
  GET  ezonline.tools/ai/upscale/09b757ac.../realesr-general-x4v3.onnx
  GET  ezonline.tools/_astro/upscale.worker-CcCGoBpe.js
  GET  ezonline.tools/ai/upscale/ort-1.21.0/ort.wasm.min.mjs
  GET  ezonline.tools/ai/upscale/ort-1.21.0/ort-wasm-simd-threaded.mjs
  GET  ezonline.tools/ai/upscale/ort-1.21.0/ort-wasm-simd-threaded.wasm

非 GET 請求數: 5
    POST 0   google-analytics.com/g/collect        （空 payload 的分析事件）
    POST 843 ezonline.tools/cdn-cgi/rum              （Cloudflare 遙測）
    HEAD 0   × 3                                    （模型下載速度競速）
payload > 1KB 的請求數: 0
```

**結論：原始工具不會上傳圖片。** 圖片資料是透過 `postMessage` 交給 Web Worker
處理，全程留在瀏覽器記憶體裡；所有 POST 的 payload 都不到 1 KB，而且都是網站自己的
分析遙測，與圖片無關。

但原始工具**首次使用必須連網**：要下載 4.9 MB 的 AI 模型（ONNX）與 ONNX Runtime
的 WASM 執行環境（12.1 MB），模型快取放在 Cache Storage / IndexedDB，
清除瀏覽器資料就沒了。

### 2.2 本版的實測結果

```
【1】載入期間沒有任何外部網路請求     ✅
【4】AI 放大期間沒有任何外部請求       ✅
【8】沒有 JS 錯誤                     ✅
```

本版把模型**整份內嵌進 HTML**，推論引擎自己寫，因此連模型下載都不需要 ——
開檔即用，離線可用，沒有快取問題。

### 2.3 離線部署實測（公司電腦情境）

針對「公司電腦沒有網路」的情境，寫了 `tests-upscale/offline_test.py` 做決定性驗證：
把瀏覽器設成**完全離線**（`context.set_offline(True)`）、用 `file://` 直接開檔，
並攔截記錄所有請求。

```
$ python3 tests-upscale/offline_test.py

【1】離線載入
  ✅ 完全離線下頁面正常載入
  ✅ 沒有嘗試任何外部網路請求
  ✅ navigator.onLine 為 false
  ✅ WebGPU 在離線下仍可取得

【2】內建相容性診斷（離線狀態下執行）
  ✅ 診斷報告產生成功
  ✅ 診斷確認可執行推論
  ✅ 報告標示模型已內嵌

【3】離線狀態下實際推論
  輸出：48x48　原始 24×24 → 放大後 48×48（2×）
  ✅ 推論在完全離線下完成
  ✅ 輸出尺寸正確（24→48）
  ✅ 輸出不是全黑（有真實模型結果）

【4】離線期間的請求總覽
  總請求數：2
    [GET] file:///…/AI-Upscale-繁體中文離線版.html
    [GET] blob:null/8652d31c-…
  ✅ 整個流程沒有任何 http/https 網路請求
  ✅ blob: 僅為本機物件網址（≤1 個，來自選檔預覽）
  ✅ 執行期無 JS 錯誤

通過 12 項，失敗 0 項
離線部署驗證通過 ✅ — 單一 HTML 在完全無網路下可完整運作
```

**結論：整個流程只有 2 個請求 —— 1 個是讀取 HTML 檔本身，1 個是 `blob:`**
（`URL.createObjectURL` 產生的**記憶體內**物件網址，指向剛選進來的圖片，
跟 EZOnline 原站上傳時產生的那個 blob 完全相同）。
**沒有任何 http/https 請求，模型不需要下載，頁面不需要連伺服器。**

### 2.4 內建相容性診斷（換電腦時用）

因為「能不能跑」取決於**該電腦的瀏覽器與顯示卡**，而不是檔案本身，
本版在介面上內建「執行相容性檢查」按鈕。拿到一台陌生電腦（公司電腦、客戶電腦）
時，按一下就會產出一份可複製的報告：

```
【AI Upscale 離線版 — 相容性檢查】

檢查時間：  2026/11/2 上午8:29:28
瀏覽器：  Mozilla/5.0 … HeadlessChrome/153.0.0.0
WebGPU：  API 存在
GPU 適配器：  取得成功
硬體資訊：  google / swiftshader
適配器類型：  軟體模擬（Fallback）— 速度會非常慢

正在跑 64×64 實測磚…
實測耗時：  186.36 秒（64×64 磚，含模型初始化）
預估 1200×900 照片：  約 2791 分鐘（偏慢）

離線狀態：  本檔案不含任何外部資源，完全離線運作
模型大小：  已內嵌於本 HTML，不需要另外下載

結論：這台電腦可以正常使用本工具
```

它會回報：瀏覽器版本、GPU 廠商與型號、**是不是軟體模擬（Fallback）**、
實測推論耗時、並依此推算一張 1200×900 照片要跑多久。
若 WebGPU 不可用，則直接列出排查步驟（瀏覽器版本、`chrome://flags`、
企業政策、以及**改用 Firefox 141+** —— 它用完全不同的 wgpu 實作，
不受 Chrome 硬體清單限制）。

---

## 三、技術棧詳述

### 3.1 一句話總結

**原生 HTML + CSS + JavaScript，零框架、零 npm 相依、零執行環境相依。**
最終產出一個 6.28 MB 的 `.html` 檔，裡面包含 AI 模型權重、ONNX 解析器、
WGSL 推論引擎與完整介面。

### 3.2 技術選型表

| 層級 | 用到的技術 | 為什麼這樣選 |
|------|-----------|-------------|
| 標記 | HTML5（`<canvas>`、`<input type=file>`、`<input type=range>`） | 瀏覽器原生提供 |
| 樣式 | CSS 自訂屬性 + Flexbox + Grid | 主題切換靠覆寫變數，不需要 Sass／PostCSS |
| 語言 | ES5 風格 JavaScript + Promise | 免編譯、免建置工具鏈 |
| 模型格式 | ONNX（`realesr-general-x4v3`） | 與原始工具完全相同的權重，輸出一致 |
| 模型解析 | **自寫 protobuf 解析器**（onnx.js，約 200 行） | 為了省掉 22.8 MB 的 ONNX Runtime |
| 推論 | **自寫 WGSL compute shader**（gpu.js） | 同上；且 GPU 執行比 WASM 快數個量級 |
| 檔案系統 | File System Access API（僅用於預覽） | 不需要 |
| 持久化 | `localStorage` + `sessionStorage` 雙寫入 | Chromium 對 `file://` 的 localStorage 寫入提交不穩定 |
| 建置 | Python 3 + `node --check` | 只做組裝與驗證，沒有 transpile |
| 測試 | Playwright（Python）+ 真實 Chromium | 端對端驗證，含「零網路請求」斷言 |

**刻意不使用**：

- ❌ **ONNX Runtime Web** —— 它的 WASM 執行環境 12.1 MB、WebGPU 版 22.8 MB，
  base64 後會讓檔案膨脹到 38 MB 以上。自己寫 shader 只要幾 KB。
- ❌ 前端框架 / 打包工具 / 任何 CDN 資源 —— 單檔離線的前提
- ❌ 外部字型 —— 系統中文字型堆疊，零外部請求

### 3.3 模型結構

原始模型的推論鏈是**線性的，沒有殘差分支**，這讓自己實作相對單純：

```
input (1×3×H×W)
  → Conv  3→64   + PRelu(64)      ← 第一層
  → Conv 64→64   + PRelu(64)  × 32 層
  → Conv 64→48                     ← 最後一層（48 = 3 × 16）
  → DepthToSpace(blocksize=4)      ← 48 通道 → 3 通道、4 倍空間
  → Add(nearest 跳接)              ← 與原圖最近鄰放大相加
  → Clip(0, 1)
```

| 項目 | 值 |
|------|-----|
| 模型檔 | `realesr-general-x4v3.onnx` |
| 大小 | 4,871,181 bytes（4.65 MB） |
| SHA-256 | `09b757accd747d7e423c1d352b3e8f23e77cc5742d04bae958d4eb8082b76fa4` |
| 來源 | HuggingFace `Heliosoph/realesrgan-onnx`（與原始工具同一份） |
| 內嵌方式 | base64 → 6,494,908 字元 → `<script type="application/octet-stream">` |
| 圖層 | 34 個 Conv + 33 個 PRelu + 1 個 DepthToSpace + 1 個 Resize + 1 個 Add + 1 個 Clip |

建置腳本會先驗證 SHA-256 與檔案大小，不符就直接讓建置失敗。

### 3.4 自寫 ONNX 解析器（onnx.js）

ONNX 就是 protobuf。這個解析器只做本模型需要的事：

- `ModelProto` → `GraphProto` → `NodeProto` 串接
- 讀出 104 個 `Constant` 節點（權重、偏置、PRelu 斜率、Resize 參數）
- 每個 `TensorProto` 的 `raw_data` 轉成 `Float32Array`
- 依節點順序組成線性推論序列

實測解析 4.85 MB 的模型只需 **75 毫秒**，權重數值與 Python 參考解析器完全一致。

### 3.5 WGSL 推論引擎（gpu.js）

三個 compute shader：

| Kernel | 內容 |
|--------|------|
| `conv3x3` | 3×3 卷積、stride 1、pad 1、dilation 1、group 1；PRelu 直接融合進 kernel（少一趟記憶體往返） |
| `depthToSpaceAddClip` | 48 通道 → 3 通道的像素重排 + nearest 跳接相加 + clamp |
| — | 跳接的 nearest 放大直接從來源緩衝取樣，不另外產生跳接緩衝 |

權重全部打包進**單一 storage buffer**，每層用 uniform buffer 帶入
`cin / cout / h / w / 權重偏移 / 偏置偏移 / 斜率偏移`。

> **踩過的坑**：一開始所有層共用同一個 uniform buffer，
> 但 `queue.writeBuffer()` 是在 command buffer 執行**之前**批次寫入的，
> 導致 34 層全部讀到最後一組參數。改成每層各有一個 uniform buffer
> 與預先建立的 bind group 才正確。

### 3.6 分塊推論

大圖直接跑會把 GPU 記憶體撐爆（一張 1600×1200 的特徵圖就要 491 MB），
所以切成 128×128 的塊，每塊往外帶 36 像素當卷積上下文：

```
區塊大小 128 × 128（自己負責的範圍）
上下文    上下左右各 36 像素（影像邊界以 clamp 取樣）
推論輸入  (128+72) × (128+72) = 200 × 200
模型輸出  800 × 800（4 倍）
拼接      每塊只取回自己負責的範圍 → 邊界沒有接縫
```

2× 輸出時，模型是 4 倍的，所以對 2×2 的區塊做平均取樣（與原始工具相同）。
Alpha 通道不參與推論，直接由原圖雙線性取樣回填，所以**透明背景完整保留**。

### 3.7 數值驗證：與 ONNX Runtime 逐像素比對

這是本專案最關鍵的一步 —— 自寫引擎的輸出必須和官方 ONNX Runtime 一致。

驗證方式：在同一個瀏覽器裡，同一張隨機測試磚（24×24）分別餵給
**ONNX Runtime Web（WASM）** 與**本專案的 WGSL 引擎**，再逐像素比對。

```
ORT 參考完成 0.3s  dims=1x3x96x96
我的引擎完成 25.7s
最大絕對差: 0.0000030   位置 22,48,1
平均絕對差: 0.000000471
最大差（0-255 尺度）: 0.0008 / 255
✓ PASS：與 ONNX Runtime 幾乎完全一致
```

**最大差異 3.0e-6**，等同於 float32 的運算雜訊 —— 兩套完全不同的實作
（ONNX Runtime 的 WASM vs. 自寫的 WGSL shader）算出來的結果一致。

> 過程中也踩到一個關鍵坑：**DepthToSpace 的通道排列**。
> 我一開始照 ONNX 規格文件的寫法實作，結果差異高達 0.75。
> 後來寫了一份純 JS 參考實作，窮舉所有通道排列與 skip 取樣方式，
> 才找出正確的是 `ci = c*16 + (Y%4)*4 + (X%4)`（像素重排的經典排列）。
> 這也證明了「先驗證再做產品」的價值 —— 如果沒有逐像素比對，
> 這個錯誤會一路混進交付檔。

### 3.8 專案結構

```
src-upscale/
  onnx.js        6,848 bytes   ONNX protobuf 解析器
  gpu.js        15,853 bytes   WGSL 推論引擎（卷積 / PRelu / DepthToSpace）
  app.js        17,244 bytes   介面、流程控制、偏好設定
  app.css       20,301 bytes   深色 + 淺色主題
  body.html     19,210 bytes   頁面骨架與繁中介面文案
build_upscale.py                組裝腳本（模型完整性驗證 + 語法檢查 + 外部資源掃描）
tests-upscale/
  e2e.py                         端對端測試（33 項）
  verify_ort.py                  數值驗證（自寫引擎 vs ONNX Runtime）
  offline_test.py                離線部署驗證（公司電腦情境，load / infer 兩階段）
  layout_test.py                 版面 / 滑桿對齊 / 下載內容（16 項）
  shot_diag.py                   產生相容性診斷面板截圖
  fixtures/                      tiny.png / alpha.png
site-ezonline/                  原始網站與模型的參考資料
  realesr-general-x4v3.onnx      模型權重（4,871,181 bytes）
  ort/                           ONNX Runtime Web（僅供數值驗證使用，交付檔不需要）
output/
  AI-Upscale-繁體中文離線版.html  ← 交付物
```

### 3.9 交付檔組成

```
AI-Upscale-繁體中文離線版.html         6584749 bytes
├─ AI 模型（base64）                    6,494,908 bytes   98.7%
├─ app.js 主程式                          16,513 bytes    0.3%
├─ gpu.js WGSL 推論引擎                  14,856 bytes    0.2%
├─ onnx.js 模型解析器                      6,684 bytes    0.1%
├─ CSS 樣式                               20,831 bytes    0.3%
├─ HTML 骨架與繁中介面內容                21,624 bytes    0.3%
└─ 主題啟動腳本                               639 bytes    0.0%
```

> 檔案體積幾乎全部是模型權重。工具本身的程式碼只有約 60 KB。

---

## 四、功能

| 功能 | 說明 |
|------|------|
| 2× / 4× 放大 | 模型原生 4 倍；選 2× 時以 2×2 區塊平均取樣 |
| 批次 | 拖放、挑選、Ctrl+V 貼上 |
| 比較滑桿 | 放大結果與原圖即時對照 |
| 輸出格式 | PNG（保留透明）或 JPG |
| 最大輸出像素 | 1600 萬（GPU）／400 萬（相容），超過時先縮小再放大 |
| 明亮／黑暗主題 | 右上角一鍵切換，選擇會被記住 |
| 零上傳 | 圖片與模型都不離開本機 |

---

## 五、與 EZOnline 線上版的差異

| 項目 | EZOnline 線上版 | 本版（完全離線） |
|------|----------------|------------------|
| 圖片是否上傳 | 否（實測確認） | **否（實測確認）** |
| 首次使用是否需下載 | 需 4.9 MB 模型 + 12.1 MB ONNX Runtime | **不用，全部內嵌** |
| 頁面分析遙測 | 有（Google Analytics、Cloudflare Insights） | **無** |
| 推論引擎 | ONNX Runtime Web（12.1–22.8 MB） | **自寫 WGSL shader，零相依** |
| 模型快取 | Cache Storage / IndexedDB，清瀏覽器資料會消失 | **不存在這問題** |
| 離線可用 | 不行 | **可以** |
| 放大 2× / 4× | 2× / 4× | 2× / 4× |
| 保留透明背景 | 有（PNG） | 有（PNG） |

模型權重使用原始工具的同一份 ONNX，因此推論結果一致。

---

## 六、測試結果

四支測試腳本，全部零失敗：

| 腳本 | 測的是什麼 | 結果 |
|------|-----------|------|
| `tests-upscale/e2e.py` | 完整使用者流程（載入 → 推論 → 下載 → 主題 → 版面） | **33 項全過** |
| `tests-upscale/layout_test.py` | 大圖版面、滑桿對齊、下載內容 | **16 項全過** |
| `tests-upscale/offline_test.py load` | 完全離線載入 + 內建診斷 | **9 項全過** |
| `tests-upscale/offline_test.py infer` | 完全離線下實際推論 | **6 項全過** |
| `tests-upscale/verify_ort.py` | 自寫引擎 vs ONNX Runtime 逐像素比對 | **5 項全過** |

### 6.1 版面／滑桿／下載測試（layout_test.py）

這個測試把**推論**換成合成資料（沙箱裡真實推論要 7 分鐘），
其餘全部是交付檔裡的真實程式碼與真實 CSS，並用與使用者實測相同尺寸的
1500×1001 測試圖（放大後 3000×2002）。

它抓到兩個真實 bug 並已修正：

| Bug | 症狀 | 成因 | 修法 |
|-----|------|------|------|
| **下載的 JPG 是全白** | 下載下來一片空白 | JPG 路徑先 `drawImage` 再 `fillRect` 鋪白底，順序寫反把圖蓋掉了 | 改成先鋪白底再 `drawImage` |
| **橘色滑桿跑到框外** | handle 位置與滑鼠不一致，按下去會出現在很右邊 | handle 用 `slider.style.left = pos × 畫布內部像素`（3000px），但畫布實際只顯示 1102px | 改用畫布的**顯示寬度**定位 |

修正前後的實測數值（框寬 1104 px、畫布 3000 px）：

```
修正前                          修正後
點擊 15% → handle 450.0px       點擊 15% → handle 166.7px  ✅
點擊 50% → handle 1502.0px ✗    點擊 50% → handle 553.0px  ✅
點擊 85% → handle 2551.0px ✗    點擊 85% → handle 938.3px  ✅
比較框 scrollWidth 1518 → 1102（不再溢出）
下載 JPG：全白 → 59,937 色、亮度 3~246
```

```
$ python3 tests-upscale/e2e.py

【1】載入頁面與離線性 ................ 6 項 ✅
【2】繁中介面與作者聲明 ............. 13 項 ✅
【3】WebGPU 後端偵測 ............... 1 項 ✅
【4】實際執行 AI 放大 ............... 8 項 ✅
【5】明亮／黑暗主題 ................. 4 項 ✅
【6】窄視窗版面 ..................... 1 項 ✅
【7】截圖 ........................... （產出 4 張）
【8】執行期錯誤檢查 ................. 1 項 ✅

========================================================
通過 33 項，失敗 0 項
全部通過 ✅
```

另有專門針對「公司電腦沒有網路」的離線部署測試 `tests-upscale/offline_test.py`：

```
$ python3 tests-upscale/offline_test.py load     # 離線載入 + 診斷
【1】離線載入
  ✅ 完全離線下頁面正常載入
  ✅ 沒有嘗試任何 http/https 網路請求
  ✅ navigator.onLine 為 false
  ✅ WebGPU 在離線下仍可取得
【2】內建相容性診斷
  ✅ 診斷報告產生成功
  ✅ 診斷確認可執行推論
  ✅ 報告標示模型已內嵌
  ✅ 診斷全程無 http/https 請求
  ✅ 執行期無 JS 錯誤
通過 9 項，失敗 0 項

$ python3 tests-upscale/offline_test.py infer    # 離線實際推論
【3】離線狀態下實際推論
  輸出：48x48　原始 24×24 → 放大後 48×48（2×）
  ✅ 推論在完全離線下完成
  ✅ 輸出尺寸正確（24→48）
  ✅ 輸出不是全黑（有真實模型結果）
【4】離線期間的請求總覽
  總請求數：2
    [GET] file:///…/AI-Upscale-繁體中文離線版.html
    [GET] blob:null/fcc44168-…
  ✅ 整個流程沒有任何 http/https 網路請求
  ✅ blob: 僅為本機物件網址（≤1 個，來自選檔預覽）
  ✅ 執行期無 JS 錯誤
通過 6 項，失敗 0 項
```

獨立的數值驗證腳本 `tests-upscale/verify_ort.py`：

```
$ python3 tests-upscale/verify_ort.py
模型載入：4.65 MB
ORT 參考完成 0.3s  dims=1x3x96x96
自寫引擎初始化完成
自寫引擎完成 25.7s
最大絕對差: 0.00000292  位置 22,48,1
平均絕對差: 0.00000048
最大差（0-255 尺度）: 0.0007 / 255
RESULT PASS 0.0000029206275939941406 4.825676822090625e-7

  ✅ 驗證流程完成
  ✅ 結果為 PASS
  ✅ 最大絕對差 < 0.003（float32 雜訊 level）
  ✅ 平均絕對差 < 1e-6
  ✅ 驗證過程無 JS 錯誤

通過 5 項，失敗 0 項
數值驗證通過 ✅ — 自寫引擎與 ONNX Runtime 結果一致
```

---

## 七、已知限制

- **需要 WebGPU**：Chrome / Edge 113+、Safari 26+、Firefox 141+（Windows）。
  沒有 GPU 的環境（例如某些虛擬機器或遠端桌面）會偵測不到，頁面會直接說明原因
  而不是默默失敗。**用介面上的「執行相容性檢查」可立刻確認**（見 2.4 節）。
- **沒有 WebGPU 就沒有 CPU 退路**：這是量過才敢寫的結論。用 ONNX Runtime 的 WASM
  CPU 後端實測，單一 200×200 工作磚（= 128×128 來源 + 36px 上下文）就要數分鐘，
  一張 400×300 照片（12 磚）會是**小時等級**。所以本版不做 CPU fallback ——
  與其給一個跑不動的選項，不如明確告知需要 WebGPU。
- **不支援 HEIC**：HEIC 需要額外的解碼器，為了讓檔案維持在合理大小，本版沒有內嵌。
  請先用系統工具轉成 JPG。
- **檔案 6.28 MB**：幾乎都是模型權重（98.7%）。工具本身只有約 60 KB。
  想更小的話得換一個更小的模型，畫質會明顯下降。
- **2× 是取樣而非真 2× 模型**：模型原生 4 倍，2× 由 4 倍輸出平均降取樣而來，
  畫質會比原生 2× 模型略柔。
- **AI 是推測不是還原**：模型會根據所見推斷合理細節，可能重建出原圖沒有的紋理。

---

## 八、版權與致謝

- 原始開發者：**[EZOnline](https://ezonline.tools/zh-tw/about-us/)**
- 原始工具：**[AI Upscale Image](https://ezonline.tools/zh-tw/upscale-image/)**
  — <https://ezonline.tools/zh-tw/upscale-image/>
- 移植版作者：**Built with ❤️ by 乙仔**

本工具為 EZOnline「AI 圖片放大」的完全離線重寫版，原始設計與概念版權歸 EZOnline 所有。
推論引擎（ONNX 解析器、WGSL shader、分塊與拼接策略）為本專案自行實作，
未使用 ONNX Runtime；內嵌的模型權重為 `realesr-general-x4v3`（ONNX），
與原始工具所使用者相同。所有圖片資料與推論運算皆於使用者本機瀏覽器內完成，
不會上傳至任何伺服器。

