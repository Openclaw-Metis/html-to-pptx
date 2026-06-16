# html-to-pptx 發版準備度報告

- Skill version: 2026.6.16
- Audit date: 2026-06-16
- Auditor: skill-creator-advanced release gate（`release_gate.py`）
- Lifecycle status: validated

## 2026.6.16 改版重點

依 GPT-5.5 的優化建議逐項評估後，實作值得做、且不破壞 skill 自足性的項目：

1. **產物功能 benchmark（最關鍵）**：新增 golden fixtures 與確定性 benchmark harness，實際用 pptxgenjs 建出原生 .pptx 並量測產物層指標（slide 數、native object 比例、缺圖 stop/report）。這補齊先前唯一卡住 publish 的 live paired functional benchmark。見 `references/fixture-benchmark.md`、`assets/evals/benchmark/run_benchmark.js`，結果在 `benchmark.json`。
2. **安全**：新增「不可信 HTML / inline JS」章節——預設不執行不可信 JS，優先靜態抽取圖表資料；非執行不可時於無網路、無憑證的沙箱進行。
3. **emoji 改為預設決策**：從「必須先問」改為 executor 式預設（沒指定環境就走 Segoe UI Emoji 可編輯），避免無謂阻塞。
4. **HTML→PPTX 對照表擴充**：補 `<img>`、background-image、超連結、項目清單、講者備註、z-index 圖層順序。
5. **泛化換算**：座標換算改為「先偵測畫布尺寸、再 fallback 比例換算」，不再寫死 1920×1080；字級維持 px÷2。
6. **可編輯性分級**：輸出契約加 A/B/C/D 分級與佔比。
7. **失敗時最小可交付**：stop/report 補上「診斷報告 + slide inventory + 缺漏資產清單；絕不用截圖默默替代」。
8. **環境章節去主機綁定**：pptxgenjs 路徑改為「依執行環境而定」，移除特定 host 的絕對路徑假設。
9. **文件分層**：新增 `references/qa-playbook.md`（視覺 QA + 視覺 diff）與 `references/fixture-benchmark.md`（benchmark 設計）。

> 刻意未做：GitHub Actions CI（其範例呼叫的 gate 工具不在本 repo，照抄會 fail；發版走本地 release gate）。README 另行補上。

## 機械 gate 結果（release_gate.py）

draft 與 publish 兩階段於 audit date 當日執行：

| Audit | Draft | Publish |
|---|---|---|
| format | PASS | PASS |
| structure | PASS | PASS |
| workflow_contract | PASS | PASS |
| semantics / semantic_rules | PASS | PASS |
| gate_language | PASS | PASS |
| lifecycle / lifecycle_state | PASS | PASS |
| eval_coverage / eval_quality / golden_trigger_set | PASS | PASS |
| wrapper_drift / surface_drift | PASS | PASS |
| migration_governance | PASS | PASS |
| skill_references / unreferenced_files | PASS | PASS |
| healthcheck | PASS | PASS |
| benchmark | PASS（functional） | PASS（`--require-live-benchmark`） |

Overall: **PASS（draft 與 publish）**。

## 常見錯誤與排查

完整事故對照見 `references/pptxgenjs-recipes.md`，視覺 QA 細節見 `references/qa-playbook.md`。重點：

- 設了 NODE_PATH 造成 `require` 失敗 → 不要覆寫 NODE_PATH，直接 require。
- CJK 大標行高被低估、裝飾條壓到標題 → 標題下方各留 0.15–0.25" 安全間距並用實際算繪確認。
- 小字用區塊級 `transparency` 算繪後變淡 → 改用較淺的實心 hex。
- 折線 / 長條圖端點無法逐點上色 → 改用角落文字標註或另疊形狀。
- QA 預覽出現中文 □（tofu）→ 多為 QA 容器缺字型，非檔案損毀。

## Policy / 需求檢查

- Release policy（`release_policy.yaml` 的 required_checks）涵蓋的 format / structure / workflow_contract / semantic_rules / gate_language / lifecycle / lifecycle_state / eval_coverage / eval_quality / golden_trigger_set / migration_governance 全數 PASS。
- Policy 的 block_if 條件（has_todo / missing_owner / stale_audit / lifecycle_state_invalid / high_risk_tool_without_approval_gate）均不成立：owner = Openclaw-Metis、無 TODO、audit date 與版本同日、無高風險工具（`external_side_effects: false`、`requires_secrets: false`）。
- 需求 / 政策結論採 fail-first：任一 policy gate 為 FAIL / BLOCKED 即整體 FAIL / BLOCKED，局部 PASS 不具放行效力。

## 剩餘風險與待辦

- **benchmark 範圍**：目前為確定性功能 build benchmark（gate-vs-bypass 式），非盲測多模型功能品質 benchmark；soffice/pdftoppm 的視覺 render-QA 在 runtime 沙箱執行，未納入維運主機的 benchmark 數字。
- **環境假設**：pptxgenjs / soffice / pdftoppm 預裝、無網路；前提改變須更新 SKILL.md「環境」章節。
- **像素一致性**：屬本質限制，重建版與原 HTML 有字體渲染微差，已在 output contract 明示。
