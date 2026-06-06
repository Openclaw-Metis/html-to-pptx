# html-to-pptx 遷移治理

本檔規範本 skill 在改名、棄用、合併、拆分時的相容性與證據要求。

## Rename（改名）

- slug `html-to-pptx` 為公開介面，改名前須在 repo 與 registry 同步更新 `name`、slash command、homepage 與所有交叉引用。
- 保留舊 slug 一個發版週期的相容別名，並在 release notes 標示對應關係，避免既有對話的觸發語失效。
- 改名一旦執行就必須同步更新 `references/readiness_report.md` 的版本與 audit date。

## Deprecate（棄用）

- 棄用前需指出替代路徑：從零做簡報 → `pptx`；只做 HTML 視覺 → `cc-designer`；截圖存圖 / PDF → 非本類任務。
- 棄用版本的 `skill_lifecycle.yaml` `status` 改為 `needs-maintenance` 或下線狀態，並在 description 開頭標註替代方案。
- 棄用不得直接刪檔；先標記、保留一個週期、再移除。

## Merge（合併）

- 僅在另一 skill 已涵蓋「HTML→可編輯原生 PPTX 重建」的完整流程與 QA 時才考慮合併。
- 合併時保留本檔的 pptxgenjs 地雷與事故對照（`references/pptxgenjs-recipes.md`），避免重建知識遺失。
- 合併後以單一 SKILL.md 為真實來源，舊 slug 設相容別名。

## Split（拆分）

- 本 skill 維持單一主要工作（重建既有 HTML deck 為可編輯 PPTX）。出現明顯不同交付物（例如「PPTX 自動排版生成」或「PDF 表單填寫」）時才拆出獨立 skill，並定義 handoff。
- 拆分後各 skill 的決策邊界與 negative trigger 不得重疊到互搶 routing。

## Compatibility（相容性）

- 公開介面：slug、description 觸發語、輸出為 `output/` 下的 .pptx。任一變更都視為相容性事件，需走 rename / deprecate 流程。
- 環境前提：pptxgenjs 預裝且以預設模組解析載入；soffice / pdftoppm 預裝供 QA；無網路。前提改變時須更新 SKILL.md 的「環境」章節。
- 向前相容原則：既有觸發語在一個發版週期內仍須命中。

## Migration Evidence（遷移證據）

- 每次改名 / 棄用 / 合併 / 拆分都必須留下證據：更新後的 `references/readiness_report.md`（含新版本與 audit date）、`skill_lifecycle.yaml` 狀態變更，以及 `assets/evals/evals.json` 觸發集是否需調整的判斷。
- 機械 gate（release gate `release_gate.py`）結果為發版前的最終證據，人工說明不得 override 機械 FAIL。
