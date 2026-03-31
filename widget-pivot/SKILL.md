---
name: databricks-widget-pivot
description: Databricks Lakeview ダッシュボードのピボットテーブルウィジェットのJSON構造リファレンス。ピボットテーブルの作成・編集、cubeGroupingSets設定、カスタムソート順（年代・時間帯）、カラースケール条件付き書式、multi-cellセル定義時に必ず参照する。「ピボット」「pivot」「クロス集計」「年代別×時間帯別」「カラースケール」「ヒートマップ」に関するダッシュボード作業で使用。
---

# ピボットテーブルウィジェット JSON構造リファレンス

このスキルは、Databricks Lakeview ダッシュボードでピボットテーブルウィジェットを作成するための完全なJSONリファレンス。
本番ダッシュボードの58ピボットウィジェットから抽出したルールに基づく。

ピボットはテーブルより構造が複雑で、`cubeGroupingSets` や `multi-cell` など固有の概念がある。

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
        "query": {
          "datasetName": "<データセット名>",
          "fields": [ ... ],
          "cubeGroupingSets": { ... },
          "disaggregated": true,
          "orders": [ ... ],
          "filters": [ ... ]
        }
      }
    ],
    "spec": {
      "version": 3,
      "widgetType": "pivot",
      "frame": { ... },
      "encodings": {
        "rows": [ ... ],
        "columns": [ ... ],
        "cell": { ... }
      }
    }
  }
}
```

---

## 2. テーブルとの主な違い

| 項目 | テーブル | ピボット |
|---|---|---|
| `spec.version` | `2` | `3` |
| `spec.widgetType` | `"table"` | `"pivot"` |
| `encodings` 構造 | `columns[]` | `rows[]` + `columns[]` + `cell{}` |
| `cubeGroupingSets` | なし | 必須 |
| `query.orders` | なし（通常） | よく使う |

---

## 3. 必須要素

### 3.1 position（必須）

```json
{
  "x": 0,
  "y": 2,
  "width": 12,    // ピボットは通常全幅
  "height": 18    // データ量に応じて調整（15〜25が一般的）
}
```

### 3.2 queries[].name（必須）

`"main_query"` が必須。テーブルと同じルール。

### 3.3 query.fields（必須）

ピボットの行・列・値に使う全フィールドを定義。

```json
"fields": [
  { "name": "日付", "expression": "`日付`" },
  { "name": "region", "expression": "`region`" },
  { "name": "area", "expression": "`area`" },
  { "name": "store_name", "expression": "`store_name`" },
  { "name": "通常予算", "expression": "`通常予算`" },
  { "name": "目標予算", "expression": "`目標予算`" }
]
```

### 3.4 query.cubeGroupingSets（必須・ピボット固有）

ピボットの行軸と列軸をどのフィールドで構成するかを指定する。

```json
"cubeGroupingSets": {
  "sets": [
    {
      "fieldNames": ["日付"]
    },
    {
      "fieldNames": ["region", "area", "store_id", "store_name"]
    }
  ]
}
```

ルール（58ピボット全件で確認済み）:
- `sets[0]` = `encodings.rows` のフィールド名と完全一致
- `sets[1]` = `encodings.columns` のフィールド名と完全一致
- 不一致の場合、ピボットが正しく動作しない

### 3.5 query.disaggregated（必須）

ピボットでは常に `true`。

### 3.6 spec.version（必須）

ピボットでは常に `3`（テーブルの `2` とは異なる）。

### 3.7 spec.frame（必須）

```json
"frame": {
  "showTitle": true,
  "title": "日次売上実績（当年・前年）"
}
```

### 3.8 spec.encodings（必須）

ピボットのencodingsは3つのセクションで構成される。

#### encodings.rows（行軸）

```json
"rows": [
  { "fieldName": "日付" }
]
```

#### encodings.columns（列軸）

```json
"columns": [
  { "fieldName": "region" },
  { "fieldName": "area" },
  { "fieldName": "store_id" },
  { "fieldName": "store_name", "headerHeight": 90 }
]
```

- `headerHeight`（オプション）: ヘッダーセルの高さ（px）。長いstore_nameを縦書きで表示する場合に使用。

#### encodings.cell（値セル・必須）

```json
"cell": {
  "type": "multi-cell",
  "fields": [
    {
      "fieldName": "通常予算",
      "cellType": "text",
      "format": { ... }
    },
    {
      "fieldName": "目標予算",
      "cellType": "text",
      "format": { ... }
    }
  ]
}
```

- `type`: 常に `"multi-cell"`（58ピボット全件で確認）
- `cellType`: 常に `"text"`
- `fields`: ピボットの値に表示するフィールド（1つでも `multi-cell` を使う）

---

## 4. オプション要素

### 4.1 query.orders（ソート順）

```json
"orders": [
  { "direction": "ASC", "expression": "`日付`" },
  { "direction": "ASC", "expression": "`region`" },
  { "direction": "ASC", "expression": "`store_id`" }
]
```

- `direction`: `"ASC"` or `"DESC"`
- 58ピボット中、全てで `orders` が指定されている

### 4.2 query.filters（行フィルター）

テーブルと同じ形式:

```json
"filters": [
  { "expression": "`row_type` IN ('shop')" }
]
```

### 4.3 カスタムソート順（scale.sort.by: "custom-order"）

年代や時間帯など、自然順序でソートできないカテゴリカルフィールドに使用。

```json
{
  "fieldName": "時間帯",
  "scale": {
    "type": "categorical",
    "sort": {
      "by": "custom-order",
      "orderedValues": ["morning", "afternoon", "evening", "all_shifts"]
    }
  }
}
```

カスタムソートは `encodings.rows[]` でも `encodings.columns[]` でも使用可能。

### 4.4 displayName（表示名変更）

行・列のヘッダー表示名を変更:

```json
{
  "fieldName": "年月",
  "displayName": "年月"
}
```

### 4.5 cell.fields[].format（値の書式）

テーブルの `format` と同じ形式。

#### 整数（カンマ区切り）
```json
{
  "fieldName": "売上",
  "cellType": "text",
  "format": {
    "type": "number-plain",
    "abbreviation": "none",
    "decimalPlaces": { "type": "exact", "places": 0 },
    "hideGroupSeparator": false
  }
}
```

#### パーセント表示
```json
{
  "fieldName": "前年比",
  "cellType": "text",
  "format": {
    "type": "number-percent",
    "decimalPlaces": { "type": "exact", "places": 1 },
    "hideGroupSeparator": true
  }
}
```

### 4.6 cell.fields[].style（カラースケール条件付き書式）

ピボットでは カラースケール（ヒートマップ風の色分け）が使われる。テーブルの `basic` ルールとは異なる。

#### style構造の解説

```
style
├── type: "color-scale"（ピボット固有。テーブルの "basic" とは異なる）
└── backgroundColor
    └── scale
        ├── type: "quantitative"
        ├── colorRamp
        │   ├── mode: "scheme"
        │   └── scheme: "redblue"（赤→白→青のグラデーション）
        └── domain
            ├── min: 0     （0% = 赤の最大）
            ├── mid: 1     （100% = 白）
            └── max: 2     （200% = 青の最大）
```

- `scheme: "redblue"`: 前年比100%を中心に、低いほど赤、高いほど青
- `domain.mid`: 中央値（前年比なら `1` = 100%）
- `domain.min/max`: 色の最小・最大範囲

---

## 5. ピボットの2層構造（重要）

ピボットウィジェットにはデータ取得側と表示レイアウト側の2つの設定層がある。両方を整合させないとUIに反映されない。

| 層 | 場所 | 役割 |
|---|---|---|
| データ取得側 | `query.fields`, `query.cubeGroupingSets`, `query.orders` | どのフィールドをどう集計するか |
| 表示レイアウト側 | `spec.encodings.rows`, `spec.encodings.columns`, `spec.encodings.cell` | どのフィールドを行/列/値に表示するか |

行列の入れ替え・フィールド追加時は両方を同時に更新すること。

片方だけ更新した場合の症状:
- `query` だけ更新 → UIのレイアウトが変わらない（旧配置のまま表示される）
- `spec.encodings` だけ更新 → データが正しく取得できずエラーや空表示になる

---

## 6. cubeGroupingSets チェックリスト

ピボット作成・編集時に必ず確認:

1. `cubeGroupingSets.sets[0].fieldNames` と `encodings.rows[].fieldName` が完全一致しているか
2. `cubeGroupingSets.sets[1].fieldNames` と `encodings.columns[].fieldName` が完全一致しているか
3. `cubeGroupingSets` のフィールドと `query.fields` の `name` が一致しているか
4. `cell.fields[].fieldName` が `query.fields` の `name` に含まれているか
5. `query.orders` の順序が `encodings.rows` → `encodings.columns` の順になっているか

---

ウィジェットJSON作成時の絶対ルール（format構造、operand形式等）は `/databricks-widget-json-rules` を参照。
