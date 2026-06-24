# pptxgenjs 重建配方與事故對照

只在需要時讀。這裡放 `SKILL.md` 不適合塞的長內容：可重用 helper 骨架、座標換算細節，以及本技能歷次重建踩到的雷與修法。

## 1. 座標與字級換算（1920×1080 → LAYOUT_WIDE）

LAYOUT_WIDE = 13.333" × 7.5"，對應 1920 × 1080 px。

- 吋：`inch(px) = px / 144`
- 字級（pt）：`fs(px) = px / 2`
- 字距：CSS `letter-spacing` 的 em 換算 ≈ `em × fontPx`，再除 2 得 pt，丟給 `charSpacing`
- 圓角：`rectRadius`（吋），18px ≈ `0.125`

常見值：`88px=0.611"`、`112px=0.778"`、`60px=0.417"`、`54px=0.375"`、`160px=1.111"`、`120px=0.833"`。

## 2. 可重用 helper 骨架

```js
const pptxgen = require("pptxgenjs");   // 不要設 NODE_PATH
const pres = new pptxgen();
pres.layout = "LAYOUT_WIDE";

const inch = (px) => px / 144;
const fs   = (px) => px / 2;
const FONT = "Arial";
const noLine = { type: "none" };        // 實測有效的「無邊框」

// 品牌色用無 # 的 6 碼 hex，集中宣告
const BLUE = "24569D", MAGENTA = "D71259", INK = "1A1D24", INK2 = "5B6472";

// 文字包裝：永遠帶 fontFace 與 margin:0；每次都傳新的 options 物件
function T(slide, text, opts) {
  slide.addText(text, Object.assign({ fontFace: FONT, margin: 0 }, opts));
}

// 共用 chrome 寫成 helper（頁碼、footer…）
function footer(slide, leftText, rightText) {
  slide.addShape("rect", { x: inch(112), y: 6.93, w: 13.333 - 2 * inch(112), h: 0.018,
    fill: { color: "E7EAEF" }, line: noLine });
  T(slide, leftText,  { x: inch(112), y: 6.985, w: 7.4, h: 0.3, fontSize: fs(18), color: INK2, valign: "middle" });
  T(slide, rightText, { x: 13.333 - inch(112) - 5, y: 6.985, w: 5, h: 0.3, fontSize: fs(18),
    color: INK2, align: "right", valign: "middle" });
}

// 卡片 = roundRect 底 + 絕對定位文字
function card(slide, x, y, w, h, fill) {
  slide.addShape("roundRect", { x, y, w, h, rectRadius: 0.125,
    fill: { color: fill || "FBFCFE" }, line: { color: fill || "E7EAEF", width: 1.5 } });
}
```

每頁包成 IIFE，讓 15+ 頁的腳本仍可讀：

```js
(function () {
  const s = pres.addSlide();
  s.background = { color: "FFFFFF" };
  // …逐頁重建…
  footer(s, "左側註腳", "右側品牌");
})();

pres.writeFile({ fileName: "output/<名稱>.pptx" }).then(f => console.log("WROTE:", f));
```

## 3. 裝飾形狀：N 瓣花（原生，不光柵化）

把每瓣做成橢圓，沿圓周以 `i*(360/N)` 度配置並 `rotate`；花心再疊一個小圓。

```js
function bloom(slide, cx, cy, r, pw, ph, colors, centerColor, centerDia) {
  for (let i = 0; i < colors.length; i++) {
    const ang = i * (360 / colors.length), rad = ang * Math.PI / 180;
    slide.addShape("ellipse", {
      x: cx + r * Math.sin(rad) - pw / 2,
      y: cy - r * Math.cos(rad) - ph / 2,
      w: pw, h: ph, fill: { color: colors[i] }, line: noLine, rotate: ang });
  }
  if (centerColor) slide.addShape("ellipse",
    { x: cx - centerDia / 2, y: cy - centerDia / 2, w: centerDia, h: centerDia,
      fill: { color: centerColor }, line: noLine });
}
```

同一個 helper 可同時做大封面花（多色）與頁眉小標記（單色、縮小參數）。

## 4. 圖表

```js
// 折線 / 平滑曲線（SVG line / curve 的對應）
slide.addChart(pres.ChartType.line,
  [{ name: "系列", labels: ["00","","04","","08"], values: [55,78,92,82,58] }],
  { x, y, w, h, chartColors: [BLUE], lineSize: 7, lineSmooth: true,
    lineDataSymbol: "none", showLegend: false, showTitle: false,
    showValAxis: false, showCatAxis: true,
    catGridLine: { style: "none" }, valGridLine: { style: "none" },
    valAxisMinVal: 20, valAxisMaxVal: 100,
    catAxisLineColor: "CFD6E2", catAxisLabelColor: INK2,
    catAxisLabelFontFace: FONT, catAxisLabelFontSize: fs(24) });
```

**限制：原生圖表的資料點 / 標記是整條同色，無法逐點上色。** 來源若在折線上用不同顏色強調特定點，改用角落文字標註或在圖表外另疊小圓形狀；不要試圖在 `addChart` 內把單點變色。

## 5. 漸層條（多停點）

pptxgenjs 不支援漸層。垂直漸層條改用數段純色矩形依來源 stop 順序由底到頂堆疊，最上與最下段用 `roundRect` 收圓角；整頁漸層背景則用純色底 + 半透明覆蓋矩形。

## 6. 事故對照（SAFE 組員疲勞評估系統 deck，15 頁）

| 症狀 | 根因 | 修法 |
|---|---|---|
| `node build.js` 報 `Cannot find module 'pptxgenjs'` | 跟著舊指引設了 `NODE_PATH=/usr/lib/node_modules`，蓋掉預設解析 | 不要設 `NODE_PATH`；直接 `require`。要定位就 `node -e "console.log(require.resolve('pptxgenjs'))"` |
| 封面裝飾條壓到大標底部 | 用 px÷2 估字級沒問題，但用公式估「3 行大標總高」會低估 CJK 行高（實際逼近 字級pt×1.3×行數） | 標題下方元素留 0.15–0.25" 安全間距，並在 QA 用實際算繪確認位置 |
| 頁碼「15」算繪後淡到看不見 | 對整個文字框設 `transparency` 讓細筆畫（數字）變淡 | 小字改用較淺的實心 hex（如淺粉），不要用區塊級 transparency |
| 折線圖端點無法各自上色 | 原生 chart 標記整條同色 | 改用角落文字標註強調高峰 / 谷底 |
| QA 圖出現中文 □ 的疑慮 | 取決於 QA 容器字型；本次 LibreOffice 容器其實正常顯示繁中 | 把純字型 tofu 視為環境問題；容器正常顯示時即可據此判斷外觀 |
| 標點不一致 | QA 後手改文字時把「。」打成 ASCII `.` | 保留全形標點，QA 清單納入標點檢查 |
| `addChart(pres.charts.X)` 報錯或 undefined | pptxgenjs 3.0 起 `pptx.charts` namespace 已棄用 → 用 `pptx.ChartType` | 改用 `pptx.ChartType.x`（`bar` / `doughnut` / `line`）；shape 傳字串 `"roundRect"` 仍有效（= ShapeType 值，非棄用的 `pptx.shapes`） |

## 7. QA 指令

```bash
soffice --headless --convert-to pdf --outdir working/ output/<名稱>.pptx
pdftoppm -jpeg -r 110 working/<名稱>.pdf .qa/slide   # 90–110 dpi 足以抓版面問題，細字再拉到 150
```

逐頁看 `.qa/slide-*.jpg`：先抓溢出 / 重疊 / 對比 / 標點，再處理純字型 tofu（多為環境問題）。
