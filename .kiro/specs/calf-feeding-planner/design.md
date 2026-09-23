# 乳用子牛 哺乳計画・費用計算アプリ — Design

## 1. 技術スタック

REQ-601・REQ-604 の制約（サーバー不要・ビルドステップ不要）に従い、以下の構成を採用する。

| 要素 | 採用技術 | 理由 |
|------|---------|------|
| マークアップ | HTML5 | 標準、追加依存なし |
| スタイル | CSS3（カスタムプロパティ） | フレームワーク不要、変数でテーマ管理 |
| ロジック | Vanilla JavaScript（ES2020） | ビルド不要、モジュール分割は `type="module"` で対応 |
| 外部依存 | なし | CDN・npm 不要 |

ファイルはすべて `src/` 以下に配置し、`index.html` をエントリポイントとする。

---

## 2. ファイル構成

```
calf-feeding-planner/
├── index.html          # シングルページ本体
└── src/
    ├── style.css       # 全スタイル定義
    ├── main.js         # エントリポイント（DOM初期化・イベント登録）
    ├── state.js        # アプリ状態の定義とデフォルト値
    ├── validation.js   # 入力値バリデーションロジック
    ├── calculator.js   # 計算ロジック（純粋関数）
    └── renderer.js     # DOM 更新ロジック
```

`main.js` だけが DOM に直接触れ、`validation.js` と `calculator.js` はプレーンオブジェクトを受け取って結果を返す純粋関数群とする。これにより将来のテスト追加が容易になる。

---

## 3. データモデル

### 3.1 アプリ状態（AppState）

```js
// state.js
const DEFAULT_STATE = {
  nursingDays: 60,          // 哺乳期間（日）
  concentration: 12.5,      // 調製濃度（%）
  unitPrice: 600,            // 単価（円/kg）
  stages: [
    { id: 1, startDay: 1,  endDay: 14, dailyAmount: 500 },
    { id: 2, startDay: 15, endDay: 42, dailyAmount: 600 },
    { id: 3, startDay: 43, endDay: 60, dailyAmount: 400 },
  ],
  nextStageId: 4,            // ステージID採番用カウンター
};
```

状態はモジュールスコープの単一オブジェクトとして管理する。状態を変更する関数は `state.js` にまとめ、直接の外部書き換えは禁止する。

### 3.2 Stage オブジェクト

```js
{
  id: Number,          // 内部識別子（削除・並び替え用）
  startDay: Number,    // 開始日齢（1-indexed）
  endDay: Number,      // 終了日齢
  dailyAmount: Number, // 1日哺乳量（g/頭/日）
}
```

### 3.3 ValidationResult オブジェクト

```js
{
  valid: Boolean,
  errors: {
    nursingDays: String | null,
    concentration: String | null,
    unitPrice: String | null,
    stages: [
      { id: Number, startDay: String|null, endDay: String|null, dailyAmount: String|null },
      ...
    ],
    coverage: String | null,   // ステージ全体のカバレッジエラー
  }
}
```

### 3.4 CalculationResult オブジェクト

```js
{
  totalPowderKg: Number,     // 総代用乳使用量（kg）
  costPerHead: Number,       // 1頭当たり費用（円）
  stageBreakdown: [
    {
      stageNo: Number,
      startDay: Number,
      endDay: Number,
      daysInStage: Number,
      dailyAmount: Number,
      subtotalPowderKg: Number,
    },
    ...
  ]
}
```

---

## 4. モジュール設計

### 4.1 `state.js`

責務: アプリ状態の保持・更新。

```
getState()                     → AppState のシャローコピーを返す
setNursingDays(value)          → nursingDays を更新
setConcentration(value)        → concentration を更新
setUnitPrice(value)            → unitPrice を更新
addStage()                     → 末尾に新ステージを追加（startDay は最終endDay+1、endDay は nursingDays）
removeStage(id)                → 指定IDのステージを削除（1件の場合は何もしない）
updateStage(id, field, value)  → 指定ステージのフィールドを更新
```

### 4.2 `validation.js`

責務: 入力値の検証。純粋関数。

```
validate(state) → ValidationResult
```

内部ロジック:
1. `nursingDays`: 整数かつ 1〜180 の範囲チェック
2. `concentration`: 数値かつ 1〜30 の範囲チェック
3. `unitPrice`: 数値かつ > 0 チェック
4. 各ステージ:
   - `startDay`: 整数 ≥ 1
   - `endDay`: 整数 ≥ startDay かつ ≤ nursingDays
   - `dailyAmount`: 数値 > 0
5. カバレッジチェック: 全ステージをstartDay昇順でソートし、day 1から連続してnursingDaysまで過不足なく埋まっているかを確認

### 4.3 `calculator.js`

責務: 計算ロジック。純粋関数。

```
calculate(state) → CalculationResult
```

計算式（REQ-301・302）:
```js
// ステージごと
const daysInStage = stage.endDay - stage.startDay + 1;
const subtotalPowderKg = (stage.dailyAmount * daysInStage) / 1000;

// 合計
const totalPowderKg = stageBreakdown.reduce((sum, s) => sum + s.subtotalPowderKg, 0);
const costPerHead = totalPowderKg * state.unitPrice;
```

`calculate()` は `validate()` が `valid: true` を返した場合にのみ呼び出す。

### 4.4 `renderer.js`

責務: DOM の読み書き。`state`・`validationResult`・`calculationResult` を受け取って画面を更新する。

```
renderStages(stages, errors)          → ステージ行を再描画
renderErrors(validationResult)        → 各フィールドのエラー表示を更新
renderResults(calcResult | null)      → 結果セクションを表示/非表示
renderDeleteButtons(stageCount)       → 削除ボタンの enabled/disabled を制御
```

### 4.5 `main.js`

責務: 初期化とイベントループ。

```
init()
  ├─ レンダリング初期実行（デフォルト値で画面構築）
  └─ イベントリスナー登録
       ├─ #nursing-days [input]        → setNursingDays → update()
       ├─ #concentration [input]       → setConcentration → update()
       ├─ #unit-price [input]          → setUnitPrice → update()
       ├─ #stages [input] (委譲)       → updateStage → update()
       ├─ #btn-add-stage [click]       → addStage → update()
       └─ .btn-remove-stage [click] (委譲) → removeStage → update()

update()
  ├─ state = getState()
  ├─ vr = validate(state)
  ├─ cr = vr.valid ? calculate(state) : null
  ├─ renderStages(state.stages, vr.errors)
  ├─ renderErrors(vr)
  └─ renderResults(cr)
```

`update()` は「状態が変わるたびに全体を再描画する」シンプルなサイクルを採る。ステージ数が高々20程度であるため、仮想DOMなしでも十分なパフォーマンスが得られる。

---

## 5. 画面レイアウト

```
┌─────────────────────────────────────────────┐
│  🐄 乳用子牛 哺乳計画・費用計算              │  ← ヘッダー
├─────────────────────────────────────────────┤
│  ▌ 基本設定                                 │
│   哺乳期間  [    60  ] 日  ← エラーメッセージ │
│   調製濃度  [  12.5  ] %                    │
│   代用乳単価 [   600  ] 円/kg               │
├─────────────────────────────────────────────┤
│  ▌ 哺乳ステージ                             │
│  ┌──────┬──────┬──────────┬──────┐         │
│  │ステージ│日齢(開始)│日齢(終了)│哺乳量(g/日)│削除│
│  ├──────┼──────┼──────────┼──────┤         │
│  │  1   │  1   │   14     │  500 │ 🗑 │     │
│  │  2   │  15  │   42     │  600 │ 🗑 │     │
│  │  3   │  43  │   60     │  400 │ 🗑 │     │
│  └──────┴──────┴──────────┴──────┘         │
│  [＋ ステージを追加]                         │
│  ⚠ カバレッジエラーメッセージ（あれば）       │
├─────────────────────────────────────────────┤
│  ▌ 計算結果                                 │
│   代用乳使用量  ████ kg                     │
│   1頭当たり費用 ████ 円                     │
│                                             │
│   ステージ別内訳                            │
│  ┌──────┬────────┬──────┬─────────┬───────┐│
│  │ステージ│日齢範囲 │日数  │哺乳量    │小計(kg)││
│  ├──────┼────────┼──────┼─────────┼───────┤│
│  │  1   │ 1〜14  │  14  │ 500 g/日│  7.00 ││
│  │  2   │15〜42  │  28  │ 600 g/日│ 16.80 ││
│  │  3   │43〜60  │  18  │ 400 g/日│  7.20 ││
│  └──────┴────────┴──────┴─────────┴───────┘│
└─────────────────────────────────────────────┘
```

モバイル（375px〜）では、ステージテーブルを横スクロール可能なコンテナに収める。

---

## 6. スタイル方針

- カラーパレットはCSSカスタムプロパティで定義（`--color-primary`, `--color-error` 等）
- エラー状態のフィールドは `border-color: var(--color-error)` + エラーテキストで視覚的に示す
- 結果セクションは `hidden` クラス（`display: none`）の付け外しで表示制御
- フォントは system-ui（OS標準フォント）を使用し、外部フォント読み込みなし
- ボタン・フォームのフォーカスリングを維持し、キーボード操作を確保（アクセシビリティ）

---

## 7. エラーメッセージ一覧

| フィールド | 条件 | メッセージ |
|-----------|------|-----------|
| 哺乳期間 | 空または非数値 | 「整数を入力してください」 |
| 哺乳期間 | 範囲外（1〜180以外） | 「1〜180 の整数を入力してください」 |
| 調製濃度 | 空または非数値 | 「数値を入力してください」 |
| 調製濃度 | 範囲外（1〜30以外） | 「1〜30 の数値を入力してください」 |
| 代用乳単価 | 空または非数値 | 「数値を入力してください」 |
| 代用乳単価 | 0以下 | 「0 より大きい数値を入力してください」 |
| ステージ 開始日齢 | 空または非整数 | 「整数を入力してください」 |
| ステージ 開始日齢 | 1未満 | 「1 以上の整数を入力してください」 |
| ステージ 終了日齢 | 空または非整数 | 「整数を入力してください」 |
| ステージ 終了日齢 | 開始日齢未満 | 「開始日齢以上の整数を入力してください」 |
| ステージ 終了日齢 | 哺乳期間超過 | 「哺乳期間（{n}日）以内の整数を入力してください」 |
| ステージ 哺乳量 | 空または非数値 | 「数値を入力してください」 |
| ステージ 哺乳量 | 0以下 | 「0 より大きい数値を入力してください」 |
| カバレッジ | ギャップあり | 「{n}日目から{m}日目の哺乳量が設定されていません」 |
| カバレッジ | 重複あり | 「ステージの日齢範囲が重複しています（{n}日目）」 |

---

## 8. Correctness — Property 設計

`correctness.md` に定義した8つの Property の実装方針を示す。
各 Property は `fast-check` の Arbitrary で任意の有効入力を生成し、Vitest の `it()` 内で `fc.assert()` を呼び出す。

### calculator.js の Property（PROP-01〜03）

```js
// tests/calculator.test.js
import fc from 'fast-check';
import { calculate } from '../src/calculator.js';

// PROP-01: totalPowderKg > 0
fc.assert(fc.property(validSingleStageArb, (state) => {
  return calculate(state).totalPowderKg > 0;
}));

// PROP-02: costPerHead === totalPowderKg × unitPrice
fc.assert(fc.property(validSingleStageArb, (state) => {
  const result = calculate(state);
  return Math.abs(result.costPerHead - result.totalPowderKg * state.unitPrice) < 1e-9;
}));

// PROP-03: Σ subtotalPowderKg === totalPowderKg
fc.assert(fc.property(validMultiStageArb, (state) => {
  const result = calculate(state);
  const sum = result.stageBreakdown.reduce((acc, s) => acc + s.subtotalPowderKg, 0);
  return Math.abs(sum - result.totalPowderKg) < 1e-9;
}));
```

### validation.js の Property（PROP-04〜06）

```js
// tests/validation.test.js
import fc from 'fast-check';
import { validate } from '../src/validation.js';

// PROP-04: 有効な入力 → valid: true
fc.assert(fc.property(validMultiStageArb, (state) => {
  return validate(state).valid === true;
}));

// PROP-05: nursingDays 範囲外 → valid: false
fc.assert(fc.property(invalidNursingDaysArb, (nursingDays) => {
  const state = { nursingDays, concentration: 12.5, unitPrice: 600,
    stages: [{ id: 1, startDay: 1, endDay: 60, dailyAmount: 500 }], nextStageId: 2 };
  return validate(state).valid === false;
}));

// PROP-06: 連続カバレッジ → errors.coverage === null
fc.assert(fc.property(validMultiStageArb, (state) => {
  return validate(state).errors.coverage === null;
}));
```

### state.js の Property（PROP-07〜08）

```js
// tests/state.test.js
import fc from 'fast-check';

// PROP-07: removeStage 後も stages.length >= 1
fc.assert(fc.property(
  fc.integer({ min: 0, max: 20 }),  // 呼び出し回数
  (callCount) => {
    // state をリセットして callCount 回 removeStage を呼ぶ
    return getState().stages.length >= 1;
  }
));

// PROP-08: addStage は stages.length を +1 する
fc.assert(fc.property(
  fc.integer({ min: 1, max: 10 }),  // 呼び出し回数 n
  (n) => {
    const before = getState().stages.length;
    for (let i = 0; i < n; i++) addStage();
    return getState().stages.length === before + n;
  }
));
```

---

## 9. 要件トレーサビリティ

| 要件ID | 対応モジュール / 設計要素 |
|--------|--------------------------|
| REQ-101 | `state.nursingDays`, `validation.js` 哺乳期間チェック |
| REQ-102 | `state.stages[]`, `renderer.renderStages()` |
| REQ-103 | `validation.js` カバレッジチェック, `renderer.renderErrors()` |
| REQ-104 | `state.concentration`, `validation.js` 調製濃度チェック |
| REQ-105 | `state.unitPrice`, `validation.js` 単価チェック |
| REQ-106 | `state.js` DEFAULT_STATE |
| REQ-201 | `main.update()` — 変更のたびに `validate()` 呼び出し |
| REQ-202 | `renderer.renderErrors()` — フィールド隣へエラーテキスト挿入 |
| REQ-203 | `validation.js` — 空・非数値チェック |
| REQ-204 | `validation.js` カバレッジチェック, セクション7エラーメッセージ |
| REQ-205 | `validation.js` 各ステージ endDay チェック |
| REQ-301 | `calculator.js calculate()` |
| REQ-302 | `calculator.js calculate()` |
| REQ-303 | `main.update()` — イベントごとに再計算 |
| REQ-304 | `renderer.renderResults()` — `toFixed(2)` |
| REQ-305 | `renderer.renderResults()` — `Math.round()` |
| REQ-401 | `renderer.renderResults()` サマリー部 |
| REQ-402 | `renderer.renderResults()` 内訳テーブル |
| REQ-403 | `renderer.renderResults(null)` — 結果セクション非表示 |
| REQ-501 | `#btn-add-stage`, `state.addStage()` |
| REQ-502 | `.btn-remove-stage`, `state.removeStage()` |
| REQ-503 | `renderer.renderDeleteButtons()` — 1件時は disabled |
| REQ-601 | 静的ファイルのみ、サーバー不要 |
| REQ-602 | ES2020 + 標準 DOM API（ポリフィル不要） |
| REQ-603 | CSS レスポンシブ（モバイルファースト、375px〜） |
| REQ-604 | Vanilla JS, `type="module"`, ビルド不要 |
| REQ-605 | 全ユーザー向けテキストを日本語で記述 |
