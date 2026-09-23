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

## プロパティベーステスト（PBT）について

PBT は **Spec の Correctness フェーズ**として管理します。
具体的な Property 定義・Arbitrary 設計・要件トレーサビリティはすべて `correctness.md` に記載しています。

### TDDとPBTの役割分担

| テスト種別 | 使う場面 | 管理場所 |
|-----------|---------|---------|
| 通常テスト（具体値） | 境界値・エラーメッセージ文言・デフォルト値の検証 | `tdd.md`（本ドキュメント） |
| PBT（性質ベース） | 計算の整合性・バリデーションの網羅性・不変条件 | `correctness.md` |

両者は補完関係にあり、同一の `tests/*.test.js` ファイル内に共存させる。

### fast-check のインストール

```bash
npm install --save-dev fast-check
```

テストファイル内では通常テストと PBT を `describe` ブロックで分けて記述する。

```js
describe('calculate()', () => {
  // --- 通常テスト ---
  describe('具体値テスト', () => {
    it('デフォルト3ステージで totalPowderKg が 31.00 になること', () => { ... });
  });

  // --- PBT（correctness.md の PROP-01〜03 に対応）---
  describe('プロパティテスト', () => {
    it('任意の有効入力で totalPowderKg が正の値になること', () => {
      fc.assert(fc.property(validSingleStageArb, (state) => {
        return calculate(state).totalPowderKg > 0;
      }));
    });
  });
});
```

---

## カバレッジ目標

| モジュール | 目標カバレッジ |
|-----------|--------------|
| `validation.js` | 90% 以上 |
| `calculator.js` | 100% |
| `state.js` | 80% 以上 |
