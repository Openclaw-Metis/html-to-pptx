# 視覺 QA 操作手冊

Step 5 視覺 QA 的細節與升級做法。核心：用**實際算繪**抓版面層級的錯（溢出 / 重疊 / 對比 / 標點），不追求像素一致。

## 基本 render QA（soffice + pdftoppm）

```bash
soffice --headless --convert-to pdf --outdir working/ output/<名稱>.pptx
pdftoppm -jpeg -r 110 working/<名稱>.pdf .qa/slide
```

逐頁看 `.qa/slide-*.jpg`：文字溢出、元件重疊、對比不足、全形標點是否被改成 ASCII。有問題改 build script 再重跑。

## 升級：半自動視覺 diff（可選）

當環境同時能算繪「原 HTML」與「PPTX」時，加一層 diff 幫快速抓「少一張圖 / 圖表歪掉 / 文字溢出」這類重大錯：

1. 能算繪時把原 HTML 各頁截成 PNG/JPG（需無頭瀏覽器）。
2. 用 `soffice` + `pdftoppm` 把產出的 PPTX 也轉成 JPG。
3. 比對 slide 數。
4. 產逐頁 side-by-side contact sheet（或逐頁 diff 分數）。
5. 回報：缺字、明顯溢出、對比不足、重大位移；**可接受差異**：字型渲染、漸層、emoji fallback。

> 環境界線：原 HTML 截圖需要無頭瀏覽器；若 runtime 沙箱只保證 soffice / pdftoppm 而沒有瀏覽器，就只做「PPTX 算繪 + contact sheet」，跳過「對照原 HTML」那半段，不要假設瀏覽器存在。

## 不要當成「檔案壞掉」的情況

- **tofu（□）**：多為 QA 容器缺 CJK / 彩色 emoji 字型，非檔案損毀。容器能正常顯示中文時就據此判斷外觀。
- **字型微差**：拉丁與 CJK 在 LibreOffice / PowerPoint 行高不同，屬本質差異。

## 常見 QA fail 檢查清單

- CJK 大標行高被低估、裝飾條壓到標題 → 標題下方各留 0.15–0.25" 安全間距，用實際算繪確認。
- 小字用區塊級 `transparency` 在算繪後變淡甚至消失 → 改用較淺的實心 hex。
- 折線 / 長條圖端點無法逐點上色 → 改用角落文字標註或另疊形狀。
- QA 後手改文字把「。」打成 ASCII「.」→ 保留全形標點。

fixtures 對應的預期版面見 `references/fixture-benchmark.md`。
