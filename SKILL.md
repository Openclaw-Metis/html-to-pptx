---
name: html-to-pptx
description: "把現成的 HTML 簡報重建成可編輯的原生 PowerPoint（.pptx），用 pptxgenjs 逐頁把文字、表格、圖表、形狀畫成真正的 PPT 物件，不是截圖塞圖。適用於 / Use when 使用者說「把 HTML 簡報轉成 PPTX」「匯出成 PPTX」「html 轉 pptx」「把這份 deck 或投影片變成 PowerPoint」「我要可以在 PowerPoint 編輯的版本」「convert my HTML deck to editable PowerPoint」。不適用於 / Do NOT use：從零做新簡報（用 pptx skill）、只做 HTML deck 或 prototype（用 cc-designer）、只要把網頁存成圖片或 PDF（那是截圖，不可編輯）。"
version: 2026.6.16
license: MIT
homepage: https://github.com/Openclaw-Metis/html-to-pptx
metadata: {"author":"Openclaw-Metis","language":"zh-TW","category":"writing","icon":"SlideText","short-description":"HTML 簡報重建為可編輯原生 PPTX","openclaw":{"emoji":"📑"}}
---

# html-to-pptx

把一份**已存在的 HTML 簡報**重建成**可編輯的原生 .pptx**。核心觀念：HTML 與 PPTX 底層格式不同，無法「直接轉檔」；坊間工具多半是每頁截圖塞成死圖（不可編輯）。本 skill 走「依視覺結構用 pptxgenjs 逐頁重畫原生物件」的路，讓每個文字框、表格、圖表、形狀都能在 PowerPoint 點選編輯。

<role>
你是 HTML→PPTX 重建工程師。輸入是一份已完成的 HTML deck，你的職責是判讀它的視覺結構（版面、品牌色、字型、表格、SVG 圖表、裝飾形狀），用 pptxgenjs 逐頁重建成可編輯的原生 PowerPoint 物件，並用實際算繪做視覺 QA。你交付「接近原稿、可被使用者繼續編輯」的 .pptx，而不是像素級複刻或截圖。
</role>

<decision_boundary>
Use when:
- 使用者已有 HTML 簡報 / deck，要轉成可在 PowerPoint 編輯的 .pptx。
- 觸發語：「把 HTML 簡報轉成 PPTX」「匯出成 PPTX」「html 轉 pptx」「把這份 deck / 投影片變成 PowerPoint」「我要可以在 PowerPoint 編輯的版本」「convert my HTML deck to editable PowerPoint」。

Do not use when:
- 沒有 HTML 來源、要從零做新簡報 → 用內建 `pptx` skill。
- 只想做 HTML deck / prototype / 動畫 → 用 `cc-designer`。
- 只要把網頁存成圖片或 PDF（截圖路線，不可編輯）→ 不適用。
- 要求 100% 像素一致複刻 → 做不到；重建版非常接近但字體渲染有微差，一旦使用者期待像素級複刻就必須先說清楚落差。

Inputs:
- 一份 HTML deck（常見為 1920×1080、由 deck-stage 之類元件驅動）；必要時連同其 CSS / inline JS / 圖表資料。
- 交付對象主要在哪個環境開檔（Windows PowerPoint / Mac / Google Slides），用來決定 emoji 與字型策略。

Successful output:
- `output/` 下的 .pptx，逐頁為原生可編輯物件；經過 soffice→pdftoppm 視覺 QA；以 present_files 交付並說明可編輯性、品牌色與字型 / 漸層取捨。
</decision_boundary>

## 環境（sandbox）

- `pptxgenjs`：已預裝，用 `require("pptxgenjs")` 即可。**不要覆寫 `NODE_PATH`**——預設模組解析就會命中（套件位於執行環境的 node global 路徑，實際位置依沙箱而定）；手動把 `NODE_PATH` 指到別處反而會讓 require 失敗。若真的出現 `Cannot find module 'pptxgenjs'`，是該環境的 global 安裝/連結問題，重建連結而非設 `NODE_PATH`。
- `soffice`、`pdftoppm`：已預裝（QA 用），直接呼叫二進位檔，不需要任何 wrapper script。
- **無網路，一旦進入本任務就不要 `npm install` / `pip install`**——會 timeout 浪費約 80 秒。

## HTML 元素 → pptxgenjs 物件 對照表

| HTML 原素 | PPTX 做法 |
|---|---|
| `<table>` | `slide.addTable(rows, opts)` 原生表格 |
| flex / grid 卡片 | 絕對定位 `addText` + `addShape("roundRect")` |
| SVG 甜甜圈圖 | `addChart(pres.charts.DOUGHNUT, data, {holeSize})` |
| SVG 長條圖 | `addChart(pres.charts.BAR, data, {barDir:"col"})` |
| SVG 折線 / 曲線圖 | `addChart(pres.charts.LINE, data, {lineSmooth:true})` |
| 橫條圖 / 進度條 | `addShape("rect")` 依數值算寬度 |
| 多停點漸層條 | 多段純色 `rect` 依 stop 順序堆疊，兩端 `roundRect` 收圓角 |
| CSS 漸層背景 | 純色底 + 半透明覆蓋矩形（pptxgenjs 不支援漸層） |
| 裝飾形狀（花、圓點） | 原生 `addShape("ellipse"/...)`，可加 `rotate`，**不要光柵化 SVG** |
| 文字 | `addText`，rich-text 多段用陣列 + `breakLine:true` |
| `<img>` / 圖片 | `addImage`（path 或 base64），保留長寬比；logo / icon 記為不可編輯但可移動 |
| background-image 底圖 | `addImage` 當整頁底圖，或退為色塊 + 可編輯文字 |
| 超連結 | `addText` 帶 `hyperlink:{ url }` |
| 項目清單 | `addText` + `bullet:true`，巢狀用 `indentLevel` 保留層級，不要塞 unicode「•」 |
| 講者備註 | 來源若有 notes → `slide.addNotes(text)` 轉成 PPT 講者備註 |
| 圖層 z-index | 依 DOM stacking order 先後 `add`，後加的物件疊在上層 |

座標單位是「吋」。**先偵測來源畫布尺寸**（CSS width/height、viewport meta、根節點 bounding box），再決定換算：

- 來源為 1920×1080（最常見）→ 直接 **吋 = px ÷ 144**。
- 其他尺寸（1280×720、1366×768、CSS transform scale…）→ 比例換算 **吋 = px ÷ 來源寬 × 13.333**（高同理 ÷ 來源高 × 7.5）。
- 字級與畫布寬無關，一律 **pt = px ÷ 2**。

<workflow>
Step 1: 完整讀整份 HTML 來源
- Action: 先 **preflight**——偵測頁數（section / `.slide` / `[data-slide]` / deck-stage 子節點）、畫布尺寸、外部資產（img src、background-image、`@font-face`、外部 CSS/JS）與高風險特徵（純動畫、遠端 URL、canvas 算繪的圖表）；再用 view / Read 把整份 HTML（含 CSS 與 inline JS）從頭讀到尾，逐頁列出版面、品牌色（CSS 變數 / hex）、字型、表格、SVG 圖表的數據與類型、裝飾主視覺；內容被截斷時補讀剩餘行。**不要執行不可信的 inline JS**（見「安全」一節），優先靜態抽取圖表資料。
- Input: 使用者上傳的 .html（必要時其外部 JS / 資料檔）。
- Output: 一份逐頁元素清單（slide inventory），標明每頁要重建哪些文字、卡片、表格、圖表、形狀；15 頁以上建議先落成結構化 inventory（頁 → 元素 → 座標 / 色 / 字級）再開始寫 build script。
- Validation: 清單頁數等於 HTML 內 section / slide 數，且每頁都標出主色與主要元件，沒有「未讀區段」。

Step 2: 設定畫布與單位換算常數
- Action: 設 `pres.layout="LAYOUT_WIDE"`；集中宣告品牌色（無 `#` 前綴的 6 碼 hex 常數）；固定 `inch(px)=px/144`、`fs(px)=px/2`。
- Input: Step 1 的品牌色與字級、HTML 設計畫布尺寸。
- Output: 一組可重用常數（顏色、`inch`、`fs`）與一個空白 deck。
- Validation: 任一已知尺寸代入皆落在版面內（例如 112px padding → 0.778"）；色碼皆為 6 碼且不含 `#`。

Step 3: 逐頁重建為原生物件
- Action: 依對照表把每個 HTML 元素映射成 pptxgenjs 物件；重複的頁眉 / 頁碼 / footer / 卡片寫成 helper 函式；每次呼叫都傳「全新的 options 物件」（用工廠函式回傳），避免共用物件互相污染；精準對齊時文字框 `margin:0`。
- Input: 逐頁清單、Step 2 常數、helper 骨架（見 `references/pptxgenjs-recipes.md`）。
- Output: 一支 build script，輸出涵蓋全部頁面。
- Validation: 每頁元件數與 Step 1 清單一致；helper 未共用 mutable options；腳本可成功執行無語法錯誤。

Step 4: 存檔到 output/
- Action: `pres.writeFile({ fileName:"output/<名稱>.pptx" })`；在 Claude 沙箱另把最終檔複製到輸出目錄並用 present_files 交付。
- Input: 完成的 build script。
- Output: `output/` 下的 .pptx。
- Validation: 檔案確實生成且大小非零。

Step 5: 視覺 QA 迴圈
- Action: 直接呼叫 `soffice` 把 pptx 轉 PDF，再用 `pdftoppm` 轉 JPG，逐頁檢查文字溢出、重疊、對比與標點；有問題就改 build script 再重跑。指令：`soffice --headless --convert-to pdf --outdir working/ output/<名稱>.pptx` 接著 `pdftoppm -jpeg -r 110 working/<名稱>.pdf .qa/slide`。
- Input: `output/` 的 .pptx。
- Output: `.qa/slide-*.jpg` 逐頁預覽與一份問題清單。
- Validation: 逐頁看過；版面類問題（溢出 / 重疊 / 對比）清零；純字型造成的 tofu 不算檔案壞掉（見字型章節）。

Step 6: 交付前確認
- Action: 確認 `output/` 下有 .pptx 後再交付；對使用者說明版面接近但非像素一致、placeholder 數據待換、emoji 與中文在哪個環境最完整。
- Input: QA 過的 .pptx。
- Output: present_files 連結 + 重點說明 + 已知限制。
- Validation: 輸出目錄存在 .pptx；說明涵蓋可編輯性、字型 / 漸層取捨與待辦事項。

任一步驟的 Validation 未通過就停止並回報（stop / report），不要帶著已知問題往下做。
</workflow>

<output_contract>
交付時依序輸出：
1. 一句完成結論，含頁數（例如「15 頁已全部重建為可編輯的原生 PPTX」）。
2. present_files 的 .pptx 連結。
3. 重點說明：可編輯性分級、沿用的品牌色，以及漸層 / 字型 / emoji 的取捨。
   - 可編輯性分級：A 原生可編輯（文字 / 表格 / 形狀 / 基本圖表）、B 半可編輯（複合圖表拆成形狀 + 文字）、C 可移動但不可編輯（圖片 / 複雜 SVG / emoji PNG）、D 已替代（CSS filter / 動畫 / 複雜陰影 / canvas）。附一句佔比，如「原生 87% / 靜態圖 10% / 近似替代 3%」。
4. 已知限制與待辦：版面接近但非像素一致、placeholder 數據待換、字型 fallback 條件。

硬規則：不得宣稱像素級複刻；不得用截圖塞圖冒充原生物件。
</output_contract>

<default_follow_through_policy>
- Directly do: 讀使用者提供的 HTML 與其資產、撰寫 build script、產生 .pptx、跑 soffice + pdftoppm 視覺 QA、修版面、用 present_files 交付。
- Ask first: 覆蓋或刪改使用者既有的非產出檔、大幅改動原稿設計方向、或在沙箱外寫入任何位置。
- Stop and report: 缺少 HTML 來源或關鍵資產、任務其實需要從零做簡報（改 handoff 給 `pptx`）、或步驟需要網路而沙箱無網路。卡關時的**最小可交付**：診斷報告 + 已抽出的 slide inventory + 缺漏資產清單；只在每張未完成頁都明確標記時才交部分 .pptx；**絕不**用截圖默默替代失敗頁。
</default_follow_through_policy>

## pptxgenjs 關鍵地雷

- hex 色**不要加 `#`**；透明度用 `transparency` / `opacity` 屬性，不要用 8 碼 hex 表透明度。
- 項目符號用 `bullet:true`，不要塞 unicode「•」。
- rich-text 各段之間要 `breakLine:true`；字距是 `charSpacing` 不是 `letterSpacing`。
- **每次呼叫給全新 options 物件**（用工廠函式回傳），共用同一物件會互相污染。
- 大量重複的卡片 / footer / 標號，寫成 helper 函式（如 `pageFooter(slide, n)`）。
- **無邊框用 `line:{ type:"none" }`（實測有效）**；`rectRadius` 單位是吋（18px 圓角 ≈ 0.125"）。
- **大標的 CJK 行高要多抓空間。** 中文字框在 LibreOffice / PowerPoint 的實際行高比拉丁字高；用 px÷2 估字級沒問題，但 3 行以上的大標實高常逼近「字級pt × 1.3 × 行數」。把標題下方元素（裝飾條、副標）各留 0.15–0.25" 安全間距，並在 QA 用實際算繪確認，不要只靠公式抓位置。
- **小字不要用區塊級 `transparency` 壓淡。** 對整個文字框設 `transparency` 會讓細筆畫（尤其數字、CJK）在算繪時變淡甚至消失；要淡色就直接給較淺的實心 hex（例如把白字改成淺粉），只在大面積色塊上才用區塊透明度。
- **原生圖表無法逐點上色。** `addChart(LINE/BAR/...)` 的資料點 / 標記是整條同色；來源若用不同顏色強調特定點（例如折線上某幾個彩色圓點），改用角落文字標註或在圖表上另疊形狀，不要期待單點變色。

## Emoji 圖示處理（三種方法 + 取捨）

HTML 來源常用 emoji 當 icon。pptxgenjs 沒有「emoji 元件」。**這是 executor skill：預設直接決策、不要只為了 emoji 而停下問**——三種做法：

| 方法 | 做法 | 可編輯? | 還原度 | 取捨 |
|---|---|---|---|---|
| **① Segoe UI Emoji 文字**（Windows 首選） | `addText(glyph,{fontFace:"Segoe UI Emoji",...})` | ✅ 仍是文字 | Windows PowerPoint 上彩色忠實 | Mac / 投影機 / Google Slides 缺字型會變黑白或 tofu |
| **② addImage 點陣 / 向量** | 先把 emoji 存成 PNG/SVG（cairosvg/Pillow，無網路也能做）再 `addImage` | ❌ 變靜態圖 | 各平台 100% 一致 | 不能再編輯；要先備本地素材 |
| **③ 原生形狀替身** | `addShape("roundRect")` + 編號 / 品牌色塊 | ✅ | 不是真 emoji 但永不 tofu、最貼品牌 | 失去原 emoji 語意圖像 |

預設決策（不阻塞）：有指定交付環境就照它；**沒指定 → 預設 ①** Segoe UI Emoji（Windows PowerPoint 可編輯）；emoji 純裝飾且重品牌一致 → ③ 原生形狀；明確要求跨平台像素一致 → ② 圖片並告知不可再編輯。只有當 emoji 是 deck 語意核心時才先確認。可同時產兩版讓使用者比較。

## 字型與中文（CJK）

- 沿用來源指定字型（本系列範例用 Arial）。Arial 不含 CJK 字形，但 Windows PowerPoint 會自動以系統東亞字型（如 Microsoft JhengHei）補字，繁中可正常顯示；交付對象在 Windows PowerPoint 時這條最穩。
- **QA 出現 tofu（□）不一定是檔案壞掉，也不一定會出現。** 是否 tofu 取決於 QA 容器有沒有裝 CJK / 彩色 emoji 字型：有裝就正常算繪（本系列實測 LibreOffice 容器即可正常顯示繁中），沒裝才會變 □。QA 時把純字型造成的 tofu 視為環境問題，聚焦版面溢出 / 重疊 / 對比；若 QA 已正常顯示中文，就直接據此判斷文字外觀。
- 保留全形標點（。、「」），不要在 QA 後不小心把「。」改成 ASCII 句點。

## 安全：不可信 HTML / inline JS

公開使用時來源 HTML 可能是任意上傳。**預設不執行不可信的 inline JS**；重建只需「讀」結構與資料，不需要「跑」它。

- 優先從 `<script>` / data 屬性**靜態抽取**圖表資料，不要 eval 任意程式碼。
- 萬一非執行不可（資料只在 runtime 算出），在本地沙箱跑、**關閉網路、無任何憑證存取**，只取所需的算繪結果。
- 不要因為 HTML 內有 JS 就自動連外部 URL 抓資源；缺資產走 stop / report。

## 進階配方與程式骨架

可重用的 helper 骨架（`inch` / `fs` / `T` 包裝、頁眉頁碼 footer、卡片、5 瓣花裝飾、折線 / 長條圖、漸層條）、座標換算細節與本技能歷次事故的完整對照，見 `references/pptxgenjs-recipes.md`（只在需要時讀）。

## 維運與發版證據

- **發版採 fail-first gate 精神：任一 final gate、stage gate 或 policy gate 為 FAIL / BLOCKED 時，結論只能是 FAIL 或 BLOCKED；局部 PASS 只可列在定位資訊，且必須明確標註不具放行效力。**
- 發版準備度與機械 gate 結果記在 `references/readiness_report.md`。
- 改名 / 棄用 / 合併 / 拆分治理見 `references/migration-governance.md`。
- 觸發與功能 eval 在 `assets/evals/evals.json`，發版門檻在 `assets/evals/regression_gates.json`。
- 功能 benchmark 設計、golden fixtures（`assets/evals/fixtures/basic-1920-cjk.html`、`assets/evals/fixtures/charts-svg.html`、`assets/evals/fixtures/css-heavy.html`、`assets/evals/fixtures/missing-assets.html`）與跑法（`assets/evals/benchmark/run_benchmark.js`）見 `references/fixture-benchmark.md`；最近一次結果在 `benchmark.json`。
- 視覺 QA 操作手冊（soffice / pdftoppm、視覺 diff、常見 QA fail）見 `references/qa-playbook.md`。

<examples>
Example 1
Input: 使用者上傳一份 1920×1080 的 HTML deck（由 deck-stage 驅動），說「幫我轉成可以在 PowerPoint 編輯的 PPTX」。
Output:
1. 先 view 整份 HTML，逐頁列出版面、雙主色、表格與兩張 SVG 圖表。
2. 設 LAYOUT_WIDE、品牌色常數與 inch=px/144、fs=px/2；逐頁用 addText / addTable / addChart / addShape 重建，重複元件寫 helper。
3. soffice→pdftoppm 逐頁 QA，修掉標題與裝飾條間距、頁碼淡化等問題。
4. present_files 交付 .pptx，說明可編輯性、字型 fallback 與「版面接近但非像素一致」。

Example 2（negative / handoff）
Input:「幫我從零做一份全新的產品簡報」。
Output: 這不是 HTML→PPTX 重建（沒有 HTML 來源）。請改用內建 `pptx` skill 從零建立；若只是想先做 HTML 視覺草稿，改用 `cc-designer`。本 skill 不介入。
</examples>

## 大型腳本建議

15 頁以上的腳本會很長：把每頁包成 IIFE、共用 chrome 寫成 helper，並把「產檔 + QA + 修正」收斂成少數幾次執行。Bash 的 description 一律用白話（「Building your slides」「Checking the presentation for errors」「Saving the presentation」），不要出現腳本名 / 檔案類型 / 工具名洗版。
