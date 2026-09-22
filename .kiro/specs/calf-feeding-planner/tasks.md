# 乳用子牛 哺乳計画・費用計算アプリ — Tasks

## 実装タスク一覧

---

### Task 1: プロジェクト構成の作成

**目的**: 空のファイルとディレクトリを揃え、実装の土台を作る。

- [ ] `src/` ディレクトリを作成する
- [ ] `index.html` を作成する（`<script type="module" src="src/main.js">` を含む最小HTML）
- [ ] `src/style.css` を作成する（空ファイル）
- [ ] `src/state.js` を作成する（空ファイル）
- [ ] `src/validation.js` を作成する（空ファイル）
- [ ] `src/calculator.js` を作成する（空ファイル）
- [ ] `src/renderer.js` を作成する（空ファイル）
- [ ] `src/main.js` を作成する（空ファイル）

**完了条件**: ブラウザで `index.html` を開いてコンソールエラーが出ないこと。

---

### Task 2: 状態管理モジュール（`src/state.js`）の実装

**目的**: アプリ全体の状態とその操作関数を定義する。対応要件: REQ-102, REQ-106, REQ-501, REQ-502, REQ-503

- [ ] `DEFAULT_STATE` を定義する（REQ-106 のデフォルト値を使用）
  ```
  nursingDays: 60, concentration: 12.5, unitPrice: 600,
  stages: [{id:1, startDay:1, endDay:14, dailyAmount:500},
           {id:2, startDay:15, endDay:42, dailyAmount:600},
           {id:3, startDay:43, endDay:60, dailyAmount:400}],
  nextStageId: 4
  ```
- [ ] `getState()` を実装する（状態のシャローコピーを返す）
- [ ] `setNursingDays(value)` を実装する
- [ ] `setConcentration(value)` を実装する
- [ ] `setUnitPrice(value)` を実装する
- [ ] `updateStage(id, field, value)` を実装する
- [ ] `addStage()` を実装する（末尾に新ステージを追加し `nextStageId` をインクリメント）
- [ ] `removeStage(id)` を実装する（`stages.length > 1` のときのみ削除）

**完了条件**: ブラウザのコンソールで各関数を呼び出し、状態が正しく変化すること。

---

### Task 3: バリデーションモジュール（`src/validation.js`）の実装

**目的**: 全入力値の検証ロジックを純粋関数として実装する。対応要件: REQ-201〜205

- [ ] `validate(state)` を実装し、`ValidationResult` オブジェクトを返す
- [ ] `nursingDays` の検証を実装する（空・非整数・1〜180範囲外）
- [ ] `concentration` の検証を実装する（空・非数値・1〜30範囲外）
- [ ] `unitPrice` の検証を実装する（空・非数値・0以下）
- [ ] 各ステージの `startDay` 検証を実装する（空・非整数・1未満）
- [ ] 各ステージの `endDay` 検証を実装する（空・非整数・startDay未満・nursingDays超過）
- [ ] 各ステージの `dailyAmount` 検証を実装する（空・非数値・0以下）
- [ ] カバレッジチェックを実装する
  - ステージを `startDay` 昇順でソート
  - day 1 から始まっているか確認（ギャップ検出）
  - 前ステージの `endDay + 1 === 次ステージの startDay` を確認（ギャップ・重複の検出）
  - 最終ステージの `endDay === nursingDays` を確認
- [ ] Design セクション7のエラーメッセージ文言をすべて実装する
- [ ] `valid` フラグを「全フィールドのエラーが null」の場合のみ `true` にする

**完了条件**: 各種不正値と正常値を引数に与えたとき、期待する `ValidationResult` が返ること。

---

### Task 4: 計算モジュール（`src/calculator.js`）の実装

**目的**: 代用乳使用量とコストの計算を純粋関数として実装する。対応要件: REQ-301〜305

- [ ] `calculate(state)` を実装し、`CalculationResult` オブジェクトを返す
- [ ] 各ステージの `daysInStage = endDay - startDay + 1` を計算する
- [ ] 各ステージの `subtotalPowderKg = (dailyAmount * daysInStage) / 1000` を計算する
- [ ] `totalPowderKg` をステージ小計の合算で計算する
- [ ] `costPerHead = totalPowderKg * unitPrice` を計算する
- [ ] `stageBreakdown` 配列（`stageNo`, `startDay`, `endDay`, `daysInStage`, `dailyAmount`, `subtotalPowderKg` を含む）を組み立てる

**完了条件**: デフォルト状態を渡したとき `totalPowderKg = 31.00`、`costPerHead = 18600` が返ること。
（7.00 + 16.80 + 7.20 = 31.00 kg、31.00 × 600 = 18,600 円）

---

### Task 5: HTMLマークアップ（`index.html`）の実装

**目的**: アプリの構造を定義し、JavaScriptから参照できる id/class を配置する。対応要件: REQ-401〜403, REQ-501〜503, REQ-605

- [ ] `<head>` に `charset`, `viewport`, `title`（日本語）, `style.css` リンクを設定する
- [ ] ヘッダー（`<header>`）に「🐄 乳用子牛 哺乳計画・費用計算」を配置する
- [ ] 基本設定セクション（`<section id="basic-settings">`）を作成する
  - `#nursing-days`（哺乳期間）、エラー要素 `#error-nursing-days`
  - `#concentration`（調製濃度）、エラー要素 `#error-concentration`
  - `#unit-price`（代用乳単価）、エラー要素 `#error-unit-price`
- [ ] ステージセクション（`<section id="stage-section">`）を作成する
  - テーブルヘッダー: ステージ / 開始日齢 / 終了日齢 / 哺乳量(g/日) / 削除
  - `<tbody id="stage-tbody">` （JavaScript で行を動的生成）
  - `<button id="btn-add-stage">＋ ステージを追加</button>`
  - カバレッジエラー要素 `<p id="error-coverage">`
- [ ] 結果セクション（`<section id="results" class="hidden">`）を作成する
  - `#result-total-powder`（総代用乳使用量）
  - `#result-cost-per-head`（1頭当たり費用）
  - 内訳テーブル `<table id="breakdown-table">` / `<tbody id="breakdown-tbody">`
- [ ] 結果非表示時のプレースホルダー要素 `<p id="results-placeholder">` を配置する

**完了条件**: HTMLのみで開いたとき構造が正しく表示され、コンソールエラーがないこと。

---

### Task 6: レンダラーモジュール（`src/renderer.js`）の実装

**目的**: 状態と計算結果を受け取りDOMを更新する関数群を実装する。対応要件: REQ-202, REQ-304, REQ-305, REQ-401〜403, REQ-403, REQ-503

- [ ] `renderStages(stages, stageErrors)` を実装する
  - `#stage-tbody` を再描画（既存行をクリアして作り直す）
  - 各行に `data-stage-id` 属性を付与する
  - `input` 要素には `data-field` 属性（`startDay` / `endDay` / `dailyAmount`）を付与する
  - エラーがある場合、対応セルにエラースタイルとエラーメッセージを表示する
- [ ] `renderDeleteButtons(stageCount)` を実装する
  - `stageCount === 1` のとき全削除ボタンを `disabled`、それ以外は `enabled`
- [ ] `renderErrors(validationResult)` を実装する
  - `#error-nursing-days`, `#error-concentration`, `#error-unit-price` を更新する
  - `#error-coverage` を更新する
  - 各入力フィールドにエラークラス（`has-error`）を付け外しする
- [ ] `renderResults(calcResult)` を実装する
  - `calcResult === null` のとき `#results` に `hidden` クラスを付与、`#results-placeholder` を表示
  - `calcResult` がある場合は `hidden` を除去し各値を更新する
  - `totalPowderKg.toFixed(2)` + " kg" で総量を表示する
  - `Math.round(costPerHead).toLocaleString('ja-JP')` + " 円" で費用を表示する
  - `#breakdown-tbody` をステージ内訳で再描画する

**完了条件**: サンプルデータを渡して `renderResults()` を呼び出したとき、画面の表示が期待値（31.00 kg / 18,600 円）と一致すること。

---

### Task 7: エントリポイント（`src/main.js`）の実装

**目的**: 初期化・イベント登録・更新サイクルを繋ぎ合わせる。対応要件: REQ-201, REQ-303

- [ ] `init()` 関数を実装する
  - `update()` を1回呼び出してデフォルト値で初期描画する
  - `DOMContentLoaded` で `init()` を呼び出す
- [ ] `update()` 関数を実装する
  ```
  1. getState()
  2. validate(state)
  3. validate.valid ? calculate(state) : null
  4. renderStages(stages, errors)
  5. renderErrors(validationResult)
  6. renderResults(calcResult)
  ```
- [ ] `#nursing-days` の `input` イベントで `setNursingDays` → `update()` を呼び出す
- [ ] `#concentration` の `input` イベントで `setConcentration` → `update()` を呼び出す
- [ ] `#unit-price` の `input` イベントで `setUnitPrice` → `update()` を呼び出す
- [ ] `#stage-tbody` への委譲イベント（`input`）で `updateStage` → `update()` を呼び出す
- [ ] `#btn-add-stage` の `click` イベントで `addStage` → `update()` を呼び出す
- [ ] `#stage-tbody` への委譲イベント（`click`、`.btn-remove-stage`）で `removeStage` → `update()` を呼び出す

**完了条件**: アプリを開いてデフォルト値で結果が表示され、各入力を変えると即時に結果が更新されること。

---

### Task 8: スタイル（`src/style.css`）の実装

**目的**: モバイルファーストのレスポンシブUIを整える。対応要件: REQ-603, REQ-605

- [ ] CSSカスタムプロパティを定義する
  ```css
  --color-primary: #2e7d32;
  --color-bg: #f9fafb;
  --color-surface: #ffffff;
  --color-border: #d1d5db;
  --color-error: #dc2626;
  --color-text: #111827;
  --color-text-muted: #6b7280;
  --radius: 6px;
  --shadow: 0 1px 3px rgba(0,0,0,.12);
  ```
- [ ] ベースリセットを実装する（`box-sizing: border-box`, `margin: 0`, `font-family: system-ui`）
- [ ] ヘッダーのスタイルを実装する
- [ ] セクション（カード）のスタイルを実装する（`background`, `border-radius`, `box-shadow`, `padding`）
- [ ] フォームラベル・入力フィールドのスタイルを実装する
- [ ] エラー状態のスタイルを実装する（`.has-error` → `border-color: var(--color-error)`、エラーテキスト `color: var(--color-error)`, `font-size: 0.8rem`）
- [ ] ステージテーブルのスタイルを実装する（横スクロール対応: `overflow-x: auto`）
- [ ] ボタンのスタイルを実装する（追加ボタン・削除ボタン・`disabled` 状態）
- [ ] 結果サマリーの強調スタイルを実装する（数値を大きめのフォントで表示）
- [ ] 内訳テーブルのスタイルを実装する
- [ ] `hidden` クラスを定義する（`display: none`）
- [ ] `min-width: 640px` のメディアクエリで、テーブルの横スクロールを解除するなどデスクトップ向け調整を行う

**完了条件**: 375px 幅と 1280px 幅の両方でレイアウトが崩れないこと。

---

### Task 9: 動作確認と最終調整

**目的**: 全要件を満たしていることを確認し、細かいUXを整える。

- [ ] デフォルト値でページを開き、31.00 kg / 18,600 円が表示されることを確認する
- [ ] 哺乳期間を変更したとき、結果が自動更新されることを確認する（REQ-303）
- [ ] ステージを追加・削除して、結果が正しく再計算されることを確認する（REQ-501, 502）
- [ ] ステージが1件のとき削除ボタンが無効化されることを確認する（REQ-503）
- [ ] カバレッジエラー（ギャップ・重複）が正しいメッセージで表示されることを確認する（REQ-103, 204）
- [ ] 各フィールドに不正値を入力したとき結果セクションが非表示になることを確認する（REQ-403）
- [ ] Chrome / Firefox / Safari / Edge で動作することを確認する（REQ-602）
- [ ] 375px 幅でレイアウトが崩れないことを確認する（REQ-603）
- [ ] `index.html` のみをブラウザで直接開いて動作することを確認する（REQ-601）
