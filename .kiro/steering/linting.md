---
inclusion: always
---

# リント設定ガイドライン（ESLint + Stylelint）

このプロジェクトは Vanilla JS（ES2020）+ CSS3 構成です。
ビルドステップ不要の制約（REQ-604）を維持しながら、静的解析ツールを開発用途で導入します。
リントツールは `npm run lint` で実行し、`index.html` の動作には影響しません。

---

## JavaScript リント: ESLint

### 設定ファイル

`eslint.config.js`（flat config）を使用します。

```js
// eslint.config.js
import js from '@eslint/js';

export default [
  js.configs.recommended,
  {
    files: ['src/**/*.js'],
    languageOptions: {
      ecmaVersion: 2020,
      sourceType: 'module',
      globals: {
        document: 'readonly',
        window: 'readonly',
        console: 'readonly',
      },
    },
    rules: {
      // エラーレベル（コードの正確性に関わるもの）
      'no-unused-vars': 'error',
      'no-undef': 'error',
      'no-console': ['warn', { allow: ['warn', 'error'] }],
      'eqeqeq': ['error', 'always'],         // === を強制
      'no-var': 'error',                      // var 禁止（const/let のみ）
      'prefer-const': 'error',               // 再代入なしは const を強制

      // 命名規則（naming-conventions.md と対応）
      'camelcase': ['error', { properties: 'always' }],

      // コードスタイル
      'semi': ['error', 'always'],
      'quotes': ['error', 'single'],
      'indent': ['error', 2],
      'no-trailing-spaces': 'error',
      'eol-last': ['error', 'always'],
    },
  },
  {
    // テストファイルは globals を緩める
    files: ['tests/**/*.test.js'],
    languageOptions: {
      globals: {
        describe: 'readonly',
        it: 'readonly',
        expect: 'readonly',
        beforeEach: 'readonly',
        afterEach: 'readonly',
      },
    },
  },
];
```

### インストール

```bash
npm install --save-dev eslint @eslint/js
```

---

## CSS リント: Stylelint

### 設定ファイル

`.stylelintrc.json` を使用します。

```json
{
  "extends": ["stylelint-config-standard"],
  "rules": {
    "custom-property-pattern": "^(color|space|radius|shadow|font)-[a-z][a-z0-9-]*$",
    "selector-class-pattern": "^([a-z][a-z0-9]*)(-[a-z0-9]+)*(__(([a-z][a-z0-9]*)(-[a-z0-9]+)*))?(--(([a-z][a-z0-9]*)(-[a-z0-9]+)*))?$",
    "color-no-invalid-hex": true,
    "declaration-block-no-duplicate-properties": true,
    "no-duplicate-selectors": true,
    "shorthand-property-no-redundant-values": true,
    "comment-empty-line-before": "always",
    "rule-empty-line-before": ["always", { "except": ["first-nested"] }]
  }
}
```

`selector-class-pattern` は BEM パターン（`block__element--modifier`）および `is-*` / `has-*` 状態クラスを許容します。

### インストール

```bash
npm install --save-dev stylelint stylelint-config-standard
```

---

## package.json スクリプト定義

```json
{
  "type": "module",
  "scripts": {
    "lint": "eslint src/ tests/ && stylelint src/style.css",
    "lint:js": "eslint src/ tests/",
    "lint:css": "stylelint src/style.css",
    "lint:fix": "eslint src/ tests/ --fix && stylelint src/style.css --fix",
    "test": "vitest run",
    "test:watch": "vitest"
  },
  "devDependencies": {
    "eslint": "^9.0.0",
    "@eslint/js": "^9.0.0",
    "stylelint": "^16.0.0",
    "stylelint-config-standard": "^36.0.0",
    "vitest": "^2.0.0"
  }
}
```

---

## リントの実行タイミング

| タイミング | コマンド | 説明 |
|-----------|---------|------|
| ファイル保存時（推奨） | Kiro の PostFileSave フックで自動実行 | `.js` / `.css` 保存のたびにチェック |
| 実装完了後 | `npm run lint` | 全ファイルを一括チェック |
| 自動修正 | `npm run lint:fix` | 自動修正可能なルールを一括修正 |
| CI（将来） | `npm run lint && npm test` | リント→テストを順番に実行 |

---

## 違反してはいけないルール（必須）

以下はエラーレベルのルールです。これらが出たままコードを提出しないでください。

| ルール | 意味 |
|--------|------|
| `no-unused-vars` | 使っていない変数・引数を残さない |
| `no-undef` | 未定義の変数を使わない |
| `eqeqeq` | `==` ではなく `===` を使う |
| `no-var` | `var` は使わず `const` / `let` を使う |
| `prefer-const` | 再代入しない変数は `const` にする |
| `camelcase` | 変数・関数名は camelCase にする（naming-conventions.md 参照） |
