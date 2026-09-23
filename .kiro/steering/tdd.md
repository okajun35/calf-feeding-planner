---
inclusion: always
---

# TDD（テスト駆動開発）ガイドライン

このプロジェクトでは、ロジック層（`validation.js`, `calculator.js`, `state.js`）に対してTDDを適用します。
DOM操作を含む `renderer.js` と `main.js` はテスト対象外とします。

---

## テストツール

ビルドステップなしでブラウザ・Node.js 両方で動作する **[Vitest](https://vitest.dev/)** を使用します。
ただし本プロジェクトはビルド不要（REQ-604）のため、テストはNode.js環境（`node --experimental-vm-modules`）で実行します。

テストファイルは `src/` と並列に `tests/` ディレクトリを作成して配置します。

```
calf-feeding-planner/
├── src/
│   ├── calculator.js
│   ├── validation.js
│   └── state.js
└── tests/
    ├── calculator.test.js
    ├── validation.test.js
    └── state.test.js
```

---

## Red-Green-Refactor サイクル

新しい関数や機能を追加するときは、必ず以下の順序で進めてください。

1. **Red**: 失敗するテストを先に書く（実装はまだ書かない）
2. **Green**: テストが通る最小限の実装を書く
3. **Refactor**: コードを整理する（テストが通ったまま）

```js
// ❌ 悪い例: 実装を先に書いてからテストを書く
// calculator.js を完成させてから calculator.test.js を書く

// ✅ 良い例: テストを先に書く
// tests/calculator.test.js
import { calculate } from '../src/calculator.js';

describe('calculate()', () => {
  it('デフォルト3ステージで totalPowderKg が 31.00 になること', () => {
    const state = {
      unitPrice: 600,
      stages: [
        { id: 1, startDay: 1,  endDay: 14, dailyAmount: 500 },
        { id: 2, startDay: 15, endDay: 42, dailyAmount: 600 },
        { id: 3, startDay: 43, endDay: 60, dailyAmount: 400 },
      ],
    };
    const result = calculate(state);
    expect(result.totalPowderKg).toBeCloseTo(31.00, 2);
  });
});
// → テストが Red になることを確認してから calculator.js を実装する
```

---

## テストの書き方

### describe / it の構造
- `describe` にはモジュール名または関数名を書く
- `it` には「〜すること」という日本語で期待する振る舞いを書く

```js
describe('validate()', () => {
  describe('nursingDays', () => {
    it('1〜180 の整数のとき nursingDays エラーが null であること', () => { ... });
    it('0 のとき nursingDays にエラーメッセージが入ること', () => { ... });
    it('181 のとき nursingDays にエラーメッセージが入ること', () => { ... });
    it('小数のとき nursingDays にエラーメッセージが入ること', () => { ... });
  });
});
```

### テストケースの網羅
各バリデーション関数について、以下のケースを必ずカバーする：
- 正常値（境界値を含む）
- 下限値 - 1（境界値の直下）
- 上限値 + 1（境界値の直上）
- 空文字列 / null / NaN
- 小数（整数のみ許容するフィールド）

### 純粋関数の原則
`validation.js` と `calculator.js` の関数はすべて純粋関数（同じ入力に対して常に同じ出力、副作用なし）として実装し、テスト容易性を維持する。

```js
// ✅ 良い例: 純粋関数
export function calculate(state) {
  // state を読み取り、新しいオブジェクトを返すだけ
  return { totalPowderKg, costPerHead, stageBreakdown };
}

// ❌ 悪い例: DOMや外部状態に依存する
export function calculate() {
  const days = document.getElementById('nursing-days').value; // テスト不可
}
```

---

## テスト実行コマンド

`package.json` を作成してテストスクリプトを定義します（`index.html` の動作には影響しない）。

```json
{
  "type": "module",
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest"
  },
  "devDependencies": {
    "vitest": "^2.0.0"
  }
}
```

- 実装前: `npm test` → 全テストが **Red**（失敗）であることを確認
- 実装後: `npm test` → 全テストが **Green**（成功）であることを確認
- リファクタリング後: `npm test` → **Green** を維持していることを確認

---

## プロパティベーステスト（PBT）

具体的な入力値を使う通常のテストに加え、**[fast-check](https://github.com/dubzzz/fast-check)** を使ったPBTを `calculator.js` と `validation.js` に適用します。
PBTは「任意の有効な入力に対して常に成り立つ性質」を検証するため、手動では思いつかないエッジケースを自動探索できます。

### インストール

```bash
npm install --save-dev fast-check
```

### 適用対象と検証すべき性質

#### `calculate()` — 3つの性質

```js
import fc from 'fast-check';
import { calculate } from '../src/calculator.js';

// Arbitrary: 有効な単一ステージ状態を生成
const validSingleStageState = fc.record({
  unitPrice: fc.integer({ min: 1, max: 100000 }),
  stages: fc.tuple(
    fc.record({
      id: fc.constant(1),
      startDay: fc.constant(1),
      endDay: fc.integer({ min: 1, max: 180 }),
      dailyAmount: fc.integer({ min: 1, max: 10000 }),
    })
  ).map(([stage]) => [stage]),
});

// 性質1: totalPowderKg は常に正
it('任意の有効な入力で totalPowderKg が正の値になること', () => {
  fc.assert(fc.property(validSingleStageState, (state) => {
    return calculate(state).totalPowderKg > 0;
  }));
});

// 性質2: costPerHead === totalPowderKg × unitPrice（乗算の整合性）
it('costPerHead が totalPowderKg × unitPrice に等しいこと', () => {
  fc.assert(fc.property(validSingleStageState, (state) => {
    const result = calculate(state);
    return Math.abs(result.costPerHead - result.totalPowderKg * state.unitPrice) < 1e-9;
  }));
});

// 性質3: stageBreakdown の小計合算 === totalPowderKg
it('stageBreakdown の subtotalPowderKg 合算が totalPowderKg に等しいこと', () => {
  fc.assert(fc.property(validSingleStageState, (state) => {
    const result = calculate(state);
    const sum = result.stageBreakdown.reduce((acc, s) => acc + s.subtotalPowderKg, 0);
    return Math.abs(sum - result.totalPowderKg) < 1e-9;
  }));
});
```

#### `validate()` — 2つの性質

```js
import fc from 'fast-check';
import { validate } from '../src/validation.js';

// 性質4: 有効な入力は常に valid: true
it('有効な状態を渡すと valid が true になること', () => {
  const validState = fc.record({
    nursingDays: fc.integer({ min: 1, max: 180 }),
    concentration: fc.float({ min: 1, max: 30 }),
    unitPrice: fc.integer({ min: 1, max: 100000 }),
    stages: fc.constant([
      { id: 1, startDay: 1, endDay: 30, dailyAmount: 500 },
    ]),
    nextStageId: fc.constant(2),
  });
  fc.assert(fc.property(validState, (state) => {
    return validate(state).valid === true;
  }));
});

// 性質5: nursingDays が範囲外なら常に valid: false
it('nursingDays が 1〜180 の範囲外なら valid が false になること', () => {
  const invalidNursingDays = fc.oneof(
    fc.integer({ max: 0 }),
    fc.integer({ min: 181 }),
  );
  fc.assert(fc.property(invalidNursingDays, (nursingDays) => {
    const state = {
      nursingDays,
      concentration: 12.5,
      unitPrice: 600,
      stages: [{ id: 1, startDay: 1, endDay: 60, dailyAmount: 500 }],
      nextStageId: 2,
    };
    return validate(state).valid === false;
  }));
});
```

### PBTと通常テストの使い分け

| テスト種別 | 使う場面 |
|-----------|---------|
| 通常テスト（具体値） | 境界値・エラーメッセージの文言・デフォルト値の検証 |
| PBT（性質ベース） | 計算の整合性・バリデーションの網羅性・不変条件の検証 |

両者は補完関係にあり、どちらか一方で代替するのではなく**併用**する。

---

## カバレッジ目標

| モジュール | 目標カバレッジ |
|-----------|--------------|
| `validation.js` | 90% 以上 |
| `calculator.js` | 100% |
| `state.js` | 80% 以上 |
