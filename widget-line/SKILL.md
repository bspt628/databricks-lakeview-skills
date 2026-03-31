---
name: databricks-widget-line
description: Databricks Lakeview ダッシュボードの折れ線グラフウィジェットのJSON構造リファレンス。折れ線グラフの作成・編集、複数系列の定義方法、Y軸フォーマット、スケール設定時に必ず参照する。「折れ線グラフ」「line chart」「推移グラフ」「前年比推移」「トレンド」に関するダッシュボード作業で使用。
---

# 折れ線グラフウィジェット JSON構造リファレンス

このスキルは、Databricks Lakeview ダッシュボードで折れ線グラフウィジェットを作成するための完全なJSONリファレンス。
本番ダッシュボードで実証済みのルールに基づく。

完全なテンプレートは [reference.md](reference.md) を参照。

---

## 1. JSON全体構造

```json
{
  "position": { ... },
  "widget": {
    "name": "<ウィジェット固有名>",
    "queries": [
      {
        "name": "main_query",
        "query": { ... }
      }
    ],
    "spec": {
      "version": 3,
      "widgetType": "line",
      "frame": { ... },
      "encodings": {
        "x": { ... },
        "y": { ... }
      }
    }
  }
}
```

---

## 2. テーブル・ピボットとの主な違い

| 項目 | テーブル | ピボット | 折れ線グラフ |
|---|---|---|---|
| `spec.version` | `2` | `3` | `3` |
| `spec.widgetType` | `"table"` | `"pivot"` | `"line"` |
| `encodings` 構造 | `columns[]` | `rows[]` + `columns[]` + `cell{}` | `x{}` + `y{}` |
| `disaggregated` | `true` | `true` | `false`（集計関数使用時） |
| 集計関数 | 不要 | 不要 | `AVG()`, `SUM()` 等をfields.expressionに使用 |

---

## 3. 必須要素

### 3.1 position（必須）

```json
{
  "x": 0,
  "y": 2,
  "width": 12,
  "height": 12
}
```

折れ線グラフは横幅が広い方が見やすい。`width: 12`（全幅）、`height: 8〜12` が一般的。

### 3.2 queries[].name（必須）

`"main_query"` が必須。他の文字列を使うとデータが表示されない。

### 3.3 query.disaggregated（必須・重要）

折れ線グラフで集計関数（`AVG()`, `SUM()` 等）を使う場合は `false` にする。
テーブルやピボットの `true` とは逆なので注意。

### 3.4 query.fields（必須）

X軸とY軸のフィールドを定義。Y軸に集計関数を使う場合、`expression` に `AVG()` や `SUM()` を指定し、`name` も集計関数付きにする。

```json
"fields": [
  { "name": "年月", "expression": "`年月`" },
  { "name": "avg(morning_sales_yoy)", "expression": "AVG(`morning_sales_yoy`)" },
  { "name": "avg(afternoon_sales_yoy)", "expression": "AVG(`afternoon_sales_yoy`)" }
]
```

`name` と `expression` の対応ルール:
- X軸（カテゴリ）: `name` = カラム名そのまま、`expression` = バッククォート参照
- Y軸（集計）: `name` = `"avg(カラム名)"` 等、`expression` = `"AVG(\`カラム名\`)"`
- `name` は `encodings.y.fields[].fieldName` と完全一致させる

---

## 4. encodings の構造

### 4.1 encodings.x（X軸・必須）

```json
"x": {
  "fieldName": "年月",
  "scale": { "type": "categorical" },
  "axis": { "labelAngle": 45 }
}
```

| プロパティ | 必須 | 説明 |
|---|---|---|
| `fieldName` | Yes | X軸に使うフィールド名 |
| `scale.type` | Yes | `"categorical"`（カテゴリ軸）が基本 |
| `axis.labelAngle` | No | ラベルの傾き（度数）。`45` で斜め表示 |

### 4.2 encodings.y（Y軸・必須）

#### 複数系列の場合（推奨パターン）

`y.fields` 配列で複数フィールドを指定。各フィールドが1本の線になる。

```json
"y": {
  "axis": { "hideTitle": true },
  "scale": {
    "type": "quantitative",
    "domain": { "max": 2 }
  },
  "fields": [
    { "fieldName": "avg(morning_sales_yoy)", "displayName": "morning" },
    { "fieldName": "avg(afternoon_sales_yoy)", "displayName": "afternoon" },
    { "fieldName": "avg(evening_sales_yoy)", "displayName": "evening" },
    { "fieldName": "avg(total_sales_yoy)", "displayName": "合計" }
  ],
  "format": {
    "type": "number-percent",
    "decimalPlaces": { "type": "max", "places": 2 }
  }
}
```

#### 単一系列の場合

`y.fields` ではなく `y.fieldName` を直接指定。

```json
"y": {
  "fieldName": "avg(売上前年比)",
  "scale": { "type": "quantitative" }
}
```

### 4.3 encodings.color（色分け・オプション）

データにカテゴリカラムがある場合、`color` で系列を自動分割できる。
ただし `y.fields` 配列による複数系列指定の方が、凡例名を自由に設定できるため推奨。

`y.fields` と `color` は排他的に使う（両方同時に使わない）。

---

## 5. Y軸フォーマット

### パーセント表示
```json
"format": {
  "type": "number-percent",
  "decimalPlaces": { "type": "max", "places": 2 }
}
```

### 整数表示
```json
"format": {
  "type": "number-plain",
  "abbreviation": "none",
  "decimalPlaces": { "type": "exact", "places": 0 },
  "hideGroupSeparator": false
}
```

### decimalPlaces.type の違い

| type | 動作 | 用途 |
|---|---|---|
| `"exact"` | 常に指定桁数を表示 | テーブル・ピボット |
| `"max"` | 指定桁数を上限として不要な0を省略 | グラフの軸ラベル |

---

## 6. 線の色・太さの設定（spec.mark）

### カラーパレット position 値（確認済み）

| position | 推定色 | 用途例 |
|---|---|---|
| 0 | 青 | デフォルト |
| 1 | 水色 | evening |
| 2 | 緑 | morning |
| 4 | 赤 | 合計（強調） |
| 7 | オレンジ | afternoon |

`{"themeColorType": "fontColor"}` で黒線（基準線に最適）。

### 基準線（100%水平線）の追加

1. SQLに固定値カラムを追加: `1.0 AS \`基準線\``
2. `queries[].fields` に追加: `{"name": "avg(基準線)", "expression": "AVG(\`基準線\`)"}`
3. `encodings.y.fields` に追加: `{"fieldName": "avg(基準線)"}`
4. `mark.colors` で `{"themeColorType": "fontColor"}` を指定して黒線にする

---

## 7. データ準備のパターン

### パターンA: カラム展開（y.fields で複数系列）

時間帯ごとの値を別カラムに展開。凡例名を自由に設定できる。

### パターンB: カテゴリカラム（color で系列分割）

時間帯をカラムの値として持つ。グラフの `color.fieldName` で自動分割。

---

ウィジェットJSON作成時の絶対ルール（format構造、operand形式等）は `/databricks-widget-json-rules` を参照。
