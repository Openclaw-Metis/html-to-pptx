# html-to-pptx 發版準備度報告

- Skill version: 2026.6.5
- Audit date: 2026-06-05
- Auditor: skill-creator-advanced release gate
- Lifecycle status: validated

## 本次改版重點（來自實作回饋）

依一次 15 頁 HTML deck → 可編輯 PPTX 的實作，把踩到的雷固化成有條件的規則：

1. **環境**：移除「設 `NODE_PATH=/usr/lib/node_modules`」這條會讓 `require` 失敗的舊指引，改為「不要覆寫 NODE_PATH、直接 require」。
2. **QA 指令**：移除不存在的 soffice wrapper 路徑引用，改為直接呼叫預裝 `soffice` / `pdftoppm`。
3. 新增地雷：CJK 大標行高要多留安全間距、小字不要用區塊級 `transparency`、原生圖表無法逐點上色、無邊框用 `line:{type:"none"}`、多停點漸層用堆疊色塊。
4. 字型章節：把「QA 一定會 tofu」修正為環境條件式（容器有 CJK 字型即正常）。
5. 結構合規：補上 `version` 與 `<role>` / `<decision_boundary>` / `<workflow>`（每步含 Action/Input/Output/Validation）/ `<output_contract>` / `<default_follow_through_policy>` / `<examples>`；frontmatter 移除被禁的 `cowork` 鍵與 `>` 折疊標量，cowork 資訊改放 `metadata`。
6. 新增發版證據：`assets/evals/evals.json`、`assets/evals/regression_gates.json`、`references/migration-governance.md`、`skill_lifecycle.yaml`、本報告，以及 `references/pptxgenjs-recipes.md`。

## 機械 gate 結果（draft stage）

以下為 release gate（`release_gate.py --stage draft`） 的逐項結果，於 audit date 當日執行：

| Audit | Status |
|---|---|
| format | PASS |
| structure | PASS |
| workflow_contract | PASS |
| semantics | PASS |
| lifecycle | PASS |
| lifecycle_state | PASS |
| eval_coverage | PASS |
| eval_quality | PASS |
| golden_trigger_set | PASS |
| wrapper_drift | PASS |
| migration_governance | PASS |
| surface_drift | PASS |
| skill_references | PASS |
| unreferenced_files | PASS |
| healthcheck | PASS |
| benchmark | SKIPPED（draft 階段不要求；publish 需 live paired benchmark）|

Overall（draft）: PASS

## 常見錯誤與排查（common errors / troubleshooting）

重建時最常見的錯誤與檢查點（完整事故對照見 `references/pptxgenjs-recipes.md`）：

- 設了 NODE_PATH 造成 `require` 失敗（MODULE_NOT_FOUND）→ 不要覆寫 NODE_PATH，直接 require。
- CJK 大標行高被低估、裝飾條壓到標題 → 標題下方元素各留 0.15–0.25" 安全間距並用實際算繪確認。
- 小字用區塊級 `transparency` 在算繪後變淡甚至消失 → 改用較淺的實心 hex。
- 折線 / 長條圖端點無法逐點上色 → 改用角落文字標註或另疊形狀。
- QA 後手改文字把「。」打成 ASCII「.」→ QA 清單納入標點檢查，保留全形標點。
- QA 預覽出現中文 □（tofu）→ 多為 QA 容器缺字型，非檔案損毀；容器正常顯示時即可據此判斷外觀。

## 剩餘風險與待辦

- **publish 門檻**：尚未跑 with-skill vs 舊版 snapshot 的成對 functional benchmark；publish 前需用 `release_gate.py --stage publish --require-live-benchmark --benchmark <benchmark.json>` 補齊，baseline 取舊版 SKILL.md snapshot。
- **環境假設**：sandbox 路徑（pptxgenjs 於 `~/.npm-global`、soffice/pdftoppm 預裝、無網路）若改變，須更新 SKILL.md「環境」章節。
- **像素一致性**：屬本質限制，重建版與原 HTML 有字體渲染微差，已在 output contract 明示。
