# 乳用子牛 哺乳計画・費用計算アプリ — Requirements

## Overview

乳用子牛の哺乳期間中に必要な代用乳の総重量および1頭当たりのコストを計算する、シングルページWebアプリケーション。Kiro University Challenge の最小実装として、入力・計算・表示の3機能に絞って構築する。

---

## Requirements

### 1. 哺乳計画の入力

**REQ-101**
The system SHALL provide an input field for the total nursing period (days), accepting integer values in the range 1–180 days.

**REQ-102**
The system SHALL allow the user to define a feeding schedule composed of one or more "stages", where each stage specifies:
- start day (integer, ≥ 1)
- end day (integer, ≥ start day, ≤ nursing period)
- daily milk replacer amount per calf (numeric, g/head/day, > 0)

**REQ-103**
The system SHALL ensure that the union of all stage day ranges exactly covers the full nursing period (day 1 through the last day) without gaps or overlaps, and SHALL display a validation error when this condition is not met.

**REQ-104**
The system SHALL provide an input field for milk replacer reconstitution concentration (%), accepting numeric values in the range 1–30%.

**REQ-105**
The system SHALL provide an input field for the unit price of milk replacer (yen per kg), accepting numeric values greater than 0.

**REQ-106**
The system SHALL set the following default values on initial load:
- Nursing period: 60 days
- Stage 1: day 1–14, 500 g/head/day
- Stage 2: day 15–42, 600 g/head/day
- Stage 3: day 43–60, 400 g/head/day
- Reconstitution concentration: 12.5%
- Unit price: 600 yen/kg

---

### 2. 入力値の検証

**REQ-201**
The system SHALL validate all input fields in real time (on every change) and SHALL NOT perform any calculation while any field contains an invalid value.

**REQ-202**
The system SHALL display an inline error message adjacent to each invalid field, describing the specific violation (e.g., "1〜180 の整数を入力してください").

**REQ-203**
The system SHALL reject non-numeric characters in all numeric input fields and SHALL treat empty fields as invalid.

**REQ-204**
The system SHALL validate that all stage day ranges collectively and completely cover day 1 through the nursing period with no gap and no overlap, and SHALL display a single summary error message identifying the first conflicting range when a violation is detected.

**REQ-205**
The system SHALL validate that each stage's end day does not exceed the total nursing period, and SHALL display an error on the offending stage row.

---

### 3. 哺乳量と費用の計算

**REQ-301**
The system SHALL calculate the total milk replacer powder consumption per head using the following formula:

```
total_powder_kg = Σ (daily_amount_g × days_in_stage) / 1000
```

where the summation is over all stages.

**REQ-302**
The system SHALL calculate the total cost of milk replacer per head using the following formula:

```
cost_per_head = total_powder_kg × unit_price_yen_per_kg
```

**REQ-303**
The system SHALL recalculate results automatically whenever any valid input value changes, with no explicit "Calculate" button required.

**REQ-304**
The system SHALL display the calculated total powder consumption rounded to two decimal places (kg).

**REQ-305**
The system SHALL display the calculated cost per head rounded to the nearest whole yen (integer).

---

### 4. 計算結果の表示

**REQ-401**
The system SHALL display the following summary values in a clearly labeled results section:
- Total milk replacer powder per head (kg)
- Total cost per head (yen)

**REQ-402**
The system SHALL display a per-stage breakdown table showing, for each stage:
- Stage number
- Day range (start day – end day)
- Days in stage
- Daily amount (g/head/day)
- Stage subtotal powder (kg, rounded to two decimal places)

**REQ-403**
The system SHALL hide the results section (or display a placeholder message) when any input is invalid, so that partial or incorrect results are never shown to the user.

---

### 5. 哺乳ステージの管理

**REQ-501**
The system SHALL allow the user to add a new stage row to the feeding schedule via an "ステージを追加" button.

**REQ-502**
The system SHALL allow the user to remove any stage row (except when only one stage remains) via a delete button on each stage row.

**REQ-503**
The system SHALL maintain at least one stage row in the feeding schedule at all times; the delete button SHALL be disabled when only one stage remains.

---

### 6. 非機能要件

**REQ-601**
The application SHALL run entirely in the browser as a static single-page application with no server-side processing required.

**REQ-602**
The application SHALL function correctly on the latest stable versions of Chrome, Firefox, Safari, and Edge.

**REQ-603**
The application SHALL be usable on screen widths of 375 px and above (mobile-first responsive design).

**REQ-604**
The application SHALL be implemented using HTML, CSS, and vanilla JavaScript (or a lightweight framework agreed upon during design), with no mandatory build step required to run it locally.

**REQ-605**
All user-facing text SHALL be written in Japanese.
