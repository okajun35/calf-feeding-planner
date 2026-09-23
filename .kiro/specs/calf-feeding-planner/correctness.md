# 乳用子牛 哺乳計画・費用計算アプリ — Correctness（Property-Based Testing）

## 概要

このドキュメントは Spec の Correctness フェーズとして、`requirements.md` から抽出した**テスト可能なProperty**を定義します。
各 Property は要件と1対1で対応し、`tests/` 配下の PBT コードとトレーサブルに結びつきます。

使用フレームワーク: **[fast-check](https://github.com/dubzzz/fast-check)** + **[Vitest](https://vitest.dev/)**

---

## Property 一覧

### PROP-01: 計算結果は常に正の値を持つ

**対応要件**: REQ-301  
**受入基準**: 任意の有効な入力（1ステージ以上、全フィールド有効）に対して `totalPowderKg > 0` が成立する

**必ず成立する条件**
> For any valid state, `calculate(state).totalPowderKg` is strictly greater than 0.

**入力値の範囲と前提条件**
- `unitPrice`: 1 以上の整数
- `stages`: 1件以上、各ステージの `dailyAmount > 0`、`endDay >= startDay`

**境界値および異常系**
- `dailyAmount = 1`、`daysInStage = 1`（最小値）→ `totalPowderKg = 0.001` > 0 ✓
- `dailyAmount = 10000`、`daysInStage = 180`（最大値）→ 正の値 ✓
- 異常系（invalid state）: `calculate()` は `validate()` が `valid: true` の場合のみ呼び出すため対象外

**対応実装タスク**: Task 4（calculator.js）

---

### PROP-02: costPerHead は totalPowderKg × unitPrice に等しい

**対応要件**: REQ-302  
**受入基準**: 任意の有効な入力に対して `costPerHead === totalPowderKg × unitPrice` が浮動小数点誤差の範囲内で成立する

**必ず成立する条件**
> For any valid state, `|calculate(state).costPerHead - calculate(state).totalPowderKg * state.unitPrice| < 1e-9`.

**入力値の範囲と前提条件**
- `unitPrice`: 1〜100,000 の整数
- `stages`: 有効なステージ配列（PROP-01 と同じ前提）

**境界値および異常系**
- `unitPrice = 1`（最小単価）→ `costPerHead = totalPowderKg × 1`
- `unitPrice = 100000`（大きな単価）→ 乗算結果が正しいこと
- 浮動小数点の丸め誤差を許容するため `< 1e-9` で比較する

**対応実装タスク**: Task 4（calculator.js）

---

### PROP-03: stageBreakdown 小計の合算は totalPowderKg に等しい

**対応要件**: REQ-301, REQ-402  
**受入基準**: 任意の有効な入力に対して `Σ stageBreakdown[i].subtotalPowderKg === totalPowderKg` が成立する

**必ず成立する条件**
> For any valid state, the sum of all `subtotalPowderKg` in `stageBreakdown` equals `totalPowderKg` within floating-point tolerance.

**入力値の範囲と前提条件**
- 複数ステージ（1〜10件）を含む有効な state
- 各ステージは `startDay`, `endDay`, `dailyAmount` がすべて正の整数

**境界値および異常系**
- ステージ数 = 1（最小）
- ステージ数 = 10（多数）
- 各ステージの `dailyAmount` がランダムな整数値

**対応実装タスク**: Task 4（calculator.js）

---

### PROP-04: 有効な入力は必ず valid:true を返す

**対応要件**: REQ-201  
**受入基準**: 任意の制約を満たす state を渡したとき `validate(state).valid === true` が成立する

**必ず成立する条件**
> For any state where all field values are within their specified valid ranges and stages cover day 1 through nursingDays without gaps or overlaps, `validate(state).valid === true`.

**入力値の範囲と前提条件**
- `nursingDays`: 1〜180 の整数
- `concentration`: 1〜30 の数値
- `unitPrice`: 1 以上の整数
- `stages`: 1件以上、`startDay=1` から `endDay=nursingDays` まで連続してカバー

**境界値および異常系**
- `nursingDays = 1`、ステージ1件（`startDay=1, endDay=1`）→ `valid: true`
- `nursingDays = 180`、ステージ1件（`startDay=1, endDay=180`）→ `valid: true`
- 多数ステージで完全カバー → `valid: true`

**対応実装タスク**: Task 3（validation.js）

---

### PROP-05: nursingDays が範囲外なら必ず valid:false を返す

**対応要件**: REQ-101, REQ-201  
**受入基準**: `nursingDays` が 1〜180 の範囲外（0以下または181以上）のとき `validate(state).valid === false` が成立する

**必ず成立する条件**
> For any state where `nursingDays <= 0` or `nursingDays >= 181`, `validate(state).valid === false`.

**入力値の範囲と前提条件**
- `nursingDays`: `Integer.MIN` 〜 0、または 181 〜 `Integer.MAX`
- その他フィールドは有効値に固定

**境界値および異常系**
- `nursingDays = 0` → `valid: false`
- `nursingDays = -1` → `valid: false`
- `nursingDays = 181` → `valid: false`
- `nursingDays = 1` → `valid: true`（境界の正常側、PROP-04 でカバー）

**対応実装タスク**: Task 3（validation.js）

---

### PROP-06: 有効なステージ構成では coverage エラーが null になる

**対応要件**: REQ-103, REQ-204  
**受入基準**: ステージが day 1 から `nursingDays` まで過不足なく連続してカバーするとき `validate(state).errors.coverage === null` が成立する

**必ず成立する条件**
> For any valid state where stages cover [1, nursingDays] contiguously, `validate(state).errors.coverage` is `null`.

**入力値の範囲と前提条件**
- ステージ数: 1〜10
- 連続して生成: stage[0].startDay=1, stage[i].startDay = stage[i-1].endDay+1, 最終 stage.endDay = nursingDays

**境界値および異常系**
- 1ステージで全期間カバー → `coverage: null`
- 最大 10 ステージで均等分割カバー → `coverage: null`
- ギャップあり（意図的に1日空ける）→ `coverage` にエラーメッセージが入る（逆Property）
- 重複あり（意図的に1日重ねる）→ `coverage` にエラーメッセージが入る（逆Property）

**対応実装タスク**: Task 3（validation.js）

---

### PROP-07: removeStage 後も stages.length は必ず 1 以上を保つ

**対応要件**: REQ-503  
**受入基準**: `removeStage()` を任意の回数呼び出しても `getState().stages.length >= 1` が常に成立する

**必ず成立する条件**
> For any sequence of `removeStage()` calls, `getState().stages.length >= 1` always holds.

**入力値の範囲と前提条件**
- 初期ステージ数: 1〜10
- `removeStage()` の呼び出し回数: 0〜20（ステージ数を超えてもよい）

**境界値および異常系**
- 初期1件で `removeStage()` を呼んでも 1件のまま
- 初期5件で5回連続 `removeStage()` → 1件が残る
- 初期1件で10回 `removeStage()` → 1件が残る

**対応実装タスク**: Task 4.5（state.js）

---

### PROP-08: addStage は stages.length を必ず 1 増やす

**対応要件**: REQ-501  
**受入基準**: `addStage()` を呼ぶたびに `stages.length` が正確に +1 される

**必ず成立する条件**
> For any state, calling `addStage()` once increases `stages.length` by exactly 1.

**入力値の範囲と前提条件**
- 任意の初期ステージ数（1〜10）

**境界値および異常系**
- `addStage()` を連続 n 回呼ぶと `stages.length` が n 増える
- 初期1件から10回 `addStage()` → `stages.length = 11`

**対応実装タスク**: Task 4.5（state.js）

---

## Property ↔ 要件トレーサビリティ

| Property ID | 要件ID | 対象モジュール | テストファイル |
|------------|--------|--------------|-------------|
| PROP-01 | REQ-301 | `calculator.js` | `tests/calculator.test.js` |
| PROP-02 | REQ-302 | `calculator.js` | `tests/calculator.test.js` |
| PROP-03 | REQ-301, REQ-402 | `calculator.js` | `tests/calculator.test.js` |
| PROP-04 | REQ-201 | `validation.js` | `tests/validation.test.js` |
| PROP-05 | REQ-101, REQ-201 | `validation.js` | `tests/validation.test.js` |
| PROP-06 | REQ-103, REQ-204 | `validation.js` | `tests/validation.test.js` |
| PROP-07 | REQ-503 | `state.js` | `tests/state.test.js` |
| PROP-08 | REQ-501 | `state.js` | `tests/state.test.js` |

---

## fast-check Arbitrary 設計指針

各 Property に対応する入力生成器（Arbitrary）の設計方針を示す。

### 有効な単一ステージ state

```js
const validSingleStageArb = fc.record({
  nursingDays: fc.integer({ min: 1, max: 180 }),
  concentration: fc.float({ min: 1, max: 30, noNaN: true }),
  unitPrice: fc.integer({ min: 1, max: 100000 }),
  stages: fc.integer({ min: 1, max: 180 }).map(endDay => [
    { id: 1, startDay: 1, endDay, dailyAmount: fc.sample(fc.integer({ min: 1, max: 10000 }), 1)[0] },
  ]),
  nextStageId: fc.constant(2),
});
```

### 連続カバレッジを持つ多段ステージ state

```js
// nursingDays を先に決め、それを n 分割して連続ステージを生成する
const validMultiStageArb = fc.integer({ min: 1, max: 180 }).chain(nursingDays =>
  fc.integer({ min: 1, max: Math.min(nursingDays, 10) }).map(stageCount => {
    // nursingDays を stageCount 個に均等分割してステージ配列を構築
    const stages = buildContiguousStages(nursingDays, stageCount);
    return { nursingDays, concentration: 12.5, unitPrice: 600, stages, nextStageId: stageCount + 1 };
  })
);
```

### 範囲外 nursingDays

```js
const invalidNursingDaysArb = fc.oneof(
  fc.integer({ max: 0 }),        // 0 以下
  fc.integer({ min: 181 }),      // 181 以上
);
```
