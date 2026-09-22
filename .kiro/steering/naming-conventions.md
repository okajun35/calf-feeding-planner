---
inclusion: always
---

# 命名規則（Naming Conventions）

このプロジェクトは HTML5 + CSS3 + Vanilla JavaScript（ES2020）で構成されるシングルページアプリケーションです。
以下の命名規則をすべてのファイルで一貫して適用してください。

---

## JavaScript

### 変数・関数
- **camelCase** を使用する。
- 意味がわかる英単語を使い、`data`, `info`, `temp` などの曖昧な名前は避ける。
- ブール値を返す変数・関数は `is`, `has`, `can` で始める。

```js
// ✅ 良い例
const nursingDays = 60;
const isValid = validate(state).valid;
function calculateCostPerHead(state) { ... }

// ❌ 悪い例
const d = 60;
const flag = true;
function calc(s) { ... }
```

### 定数
- **UPPER_SNAKE_CASE** を使用する。
- モジュールスコープの不変値（デフォルト値、閾値）に適用する。

```js
// ✅ 良い例
const MAX_NURSING_DAYS = 180;
const MIN_CONCENTRATION = 1;
const DEFAULT_UNIT_PRICE = 600;
```

### クラス（将来追加する場合）
- **PascalCase** を使用する。

### プライベートな内部ヘルパー関数
- アンダースコアプレフィックスは使わない。代わりにモジュールスコープに閉じ込め、`export` しない。

---

## CSS

### クラス名
- **kebab-case** を使用する。
- BEM ライクな構造（`.block`, `.block__element`, `.block--modifier`）を採用する。
- 状態クラスは `is-` または `has-` プレフィックスを付ける。

```css
/* ✅ 良い例 */
.stage-table { }
.stage-table__row { }
.stage-table__row--error { }
.input-field { }
.input-field.has-error { }
.btn-add-stage { }

/* ❌ 悪い例 */
.stageTable { }
.StageTable { }
.addBtn { }
```

### CSSカスタムプロパティ
- `--` プレフィックスの後に **kebab-case** で記述する。
- カテゴリを先頭に置く（`--color-*`, `--space-*`, `--radius-*`）。

```css
/* ✅ 良い例 */
:root {
  --color-primary: #2e7d32;
  --color-error: #dc2626;
  --space-md: 1rem;
  --radius-base: 6px;
}
```

---

## HTML

### id属性
- **kebab-case** を使用する。
- JavaScriptから参照する要素は必ず `id` を付与し、クラスセレクタで取得しない。

```html
<!-- ✅ 良い例 -->
<input id="nursing-days" type="number">
<p id="error-nursing-days" class="error-text"></p>
<tbody id="stage-tbody"></tbody>
<button id="btn-add-stage">＋ ステージを追加</button>

<!-- ❌ 悪い例 -->
<input id="nursingDays" type="number">
<input id="NursingDays" type="number">
```

### data属性
- **kebab-case** を使用する。
- 動的生成する要素の識別子は `data-*` 属性で持つ。

```html
<!-- ✅ 良い例 -->
<tr data-stage-id="1">
  <input data-field="start-day" type="number">
</tr>
```

---

## ファイル名
- **kebab-case** を使用する（`main.js`, `validation.js`, `style.css`）。
- テストファイルは対象ファイル名に `.test.js` を付ける（`validation.test.js`）。
