# 功能 benchmark：golden fixtures 與跑法

本檔說明 html-to-pptx 的**確定性功能 benchmark**——用一組 golden HTML deck 實際建出原生 .pptx，量測產物層指標，證明「重建成可編輯原生物件」確實有效，並作為 publish gate 的 benchmark 證據。

> 與 routing eval（`assets/evals/evals.json`）互補：那邊測「會不會被正確觸發」，這邊測「產物品質」。

## Golden fixtures

| Fixture | 目的 | 重點 |
|---|---|---|
| `assets/evals/fixtures/basic-1920-cjk.html` | 1920×1080、繁中、表格、頁碼 | 字型 fallback、全形標點、slide 數一致 |
| `assets/evals/fixtures/charts-svg.html` | SVG bar / donut | 轉原生 `addChart`（非截圖） |
| `assets/evals/fixtures/css-heavy.html` | grid / flex 卡片、圓角、漸層、陰影 | `roundRect` + 文字、漸層退色塊取捨 |
| `assets/evals/fixtures/missing-assets.html` | 缺本地圖檔 | 必須 stop / report，**不可**默默捏造或塞截圖 |

## 量測指標（產物層）

每個 fixture 量測：`pptx_exists`、`pptx_nonzero`、`slide_count_match`、`native_object_ratio`、`missing_asset_handled`。
`native_rebuild` 單一 fixture 通過條件：產出非空 .pptx、頁數一致、`native_object_ratio ≥ 0.9`；`missing-assets` 的通過條件改為「正確偵測並 stop / report」。

## 兩個配置（primary 在前、baseline 在後）

- `native_rebuild`（primary）：本 skill 的正路——逐頁重建原生可編輯物件。
- `screenshot_dump`（baseline）：本 skill 要取代的反模式——截圖塞圖，可編輯物件數為 0，`pass_rate` 依定義為 0。

此對比沿用 skill-creator-advanced 自家 release benchmark 的 gate-vs-bypass 模式：確定性、可重跑、誠實標註，非盲測多模型功能品質 benchmark。

## 跑法

```bash
node assets/evals/benchmark/run_benchmark.js \
  --out benchmark.json \
  --skill-version <SKILL.md 的 version> \
  --git-commit "$(git rev-parse --short HEAD)"
```

- 產物 .pptx 與逐項明細寫到沙箱外的工作目錄（預設 `/tmp/htp-bench`），**不落進 `assets/`**。
- 受版本門檻約束的 `benchmark.json` 寫到 skill 根目錄，作為 publish gate 證據；`metadata.skill_version` 必須等於 SKILL.md 的 `version`。

## 門檻

回歸門檻在 `assets/evals/regression_gates.json`：`pass_rate` delta 不得為負、時間/與 token 增量在上限內。primary 對 baseline 的 `pass_rate` 必須非負。

## 注意：render-QA 的環境界線

soffice / pdftoppm 的視覺算繪 QA（見 `references/qa-playbook.md`）在 skill 的 runtime 沙箱跑；維運主機若未安裝這兩個二進位檔，render-QA 在該主機不納入本 benchmark 數字，延到 runtime 沙箱執行。
