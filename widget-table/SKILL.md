---
name: databricks-widget-table
description: Databricks Lakeview ダッシュボードのテーブルウィジェットのJSON構造リファレンス。テーブルウィジェットの作成・編集、カラム書式設定、条件付き書式（前年比の赤字表示など）、displayName設定時に必ず参照する。「テーブル」「一覧表」「table widget」「条件付き書式」「前年比を赤くしたい」に関するダッシュボード作業で使用。
---

# テーブルウィジェット JSON構造リファレンス

このスキルは、Databricks Lakeview ダッシュボードでテーブルウィジェットを作成するための完全なJSONリファレンス。
本番ダッシュボードの21テーブルウィジェットから抽出したルールに基づく。

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
      "version": 2,
      "widgetType": "table",
      "frame": { ... },
      "encodings": {
        "columns": [ ... ]
      }
    }
  }
}
```

---

## 2. 必須要素

### 2.1 position（必須）

```json
{
  "x": 0,        // 0-11（12カラムグリッド）
  "y": 2,        // フィルターの下に配置する場合は2以上
  "width": 12,   // テーブルは通常width=12（全幅）
  "height": 15   // データ量に応じて調整（10〜20が一般的）
}
```

### 2.2 widget.name（必須）

ウィジェットの識別名。自由な文字列。

```
"name": "pivot_daily_sales"
```

### 2.3 queries[].name（必須・重要）

**テーブルウィジェットでは `"main_query"` が必須**。他の文字列を使うとデータが表示されない。

```json
"queries": [
  {
    "name": "main_query",
    ...
  }
]
```

これは9つのダッシュボード・137ウィジェットの調査で実証済み。

### 2.4 query.datasetName（必須）

```
"datasetName": "ds_02_daily_sales_combined"
```

`serialized_dashboard.datasets` に定義されたキーと一致させる。

### 2.5 query.fields（必須）

テーブルに表示するカラムを定義。

```json
"fields": [
  {
    "name": "region",
    "expression": "`region`"
  },
  {
    "name": "売上",
    "expression": "`売上`"
  }
]
```

- `name`: フィールドの識別名
- `expression`: SQL式。カラム参照はバッククォートで囲む

### 2.6 query.disaggregated（必須）

**テーブルでは常に `true`**（フィルターの `false` とは逆）。

```json
"disaggregated": true
```

### 2.7 spec.version（必須）

テーブルは常に `2`。

### 2.8 spec.frame（必須）

```json
"frame": {
  "showTitle": true,
  "title": "日次売上実績"
}
```

全21テーブルで `showTitle: true` が設定されている。

### 2.9 spec.encodings.columns（必須）

テーブルの列定義。`query.fields` で定義したフィールドのうち、表示したいものを列挙する。

```json
"encodings": {
  "columns": [
    { "fieldName": "region" },
    { "fieldName": "area" },
    { "fieldName": "store_name" },
    { "fieldName": "売上", "format": { ... } }
  ]
}
```

`fieldName` は `query.fields[].name` と一致させる。

---

## 3. オプション要素

### 3.1 query.filters（行フィルター）

特定の行だけ表示したい場合。SQL式で条件を指定。

```json
"filters": [
  {
    "expression": "`row_type` IN ('shop')"
  }
]
```

21テーブル中8テーブルで使用。主にサマリー行とディテール行を分けるために使われる。

### 3.2 columns[].format（数値書式）

#### 整数（カンマ区切り）
```json
{
  "fieldName": "売上",
  "format": {
    "type": "number-plain",
    "abbreviation": "none",
    "decimalPlaces": {
      "type": "exact",
      "places": 0
    },
    "hideGroupSeparator": false
  }
}
```

#### パーセント表示
```json
{
  "fieldName": "前年比売上",
  "format": {
    "type": "number-percent",
    "decimalPlaces": {
      "type": "exact",
      "places": 1
    },
    "hideGroupSeparator": true
  }
}
```

#### format.type の選択肢（実績あり）

| type | 用途 | 例 |
|---|---|---|
| `number-plain` | 通常の数値 | 1,234,567 |
| `number-percent` | パーセント表示 | 98.5% |

#### format のプロパティ一覧

| プロパティ | 型 | 必須 | 説明 |
|---|---|---|---|
| `type` | string | Yes | `number-plain` or `number-percent` |
| `abbreviation` | string | No | `"none"` で省略なし（number-plainのみ） |
| `decimalPlaces.type` | string | Yes | `"exact"` |
| `decimalPlaces.places` | number | Yes | 小数点以下桁数 |
| `hideGroupSeparator` | boolean | No | `true` で3桁区切りなし |

### 3.3 columns[].displayName（表示名変更）

カラム名と表示ラベルを別にしたい場合:

```json
{
  "fieldName": "前年比人数",
  "displayName": "前年比客数",
  "format": { ... }
}
```

### 3.4 columns[].style（条件付き書式）

値に応じてセルの文字色・背景色を変更する。

#### 前年比 < 100%（1未満）を赤字にする例

```json
{
  "fieldName": "前年比売上",
  "format": {
    "type": "number-percent",
    "decimalPlaces": { "type": "exact", "places": 1 },
    "hideGroupSeparator": true
  },
  "style": {
    "type": "basic",
    "rules": [
      {
        "condition": {
          "operand": {
            "type": "data-value",
            "value": "1"
          },
          "operator": "<"
        },
        "foregroundColor": {
          "themeColorType": "visualizationColors",
          "position": 4
        }
      }
    ]
  }
}
```

#### style の構造

```
style
├── type: "basic"（現在確認されている唯一のタイプ）
└── rules: [
      {
        condition
        ├── operand
        │   ├── type: "data-value"
        │   └── value: "1"（比較値、文字列で指定）
        ├── operator: "<" | ">" | "<=" | ">=" | "==" | "!="
        foregroundColor（文字色）| backgroundColor（背景色）
        ├── themeColorType: "visualizationColors"
        └── position: 4（カラーパレットのインデックス、4=赤系）
      }
    ]
```

#### カラーパレット position 値（実績確認済み）

| position | 色 | 用途例 |
|---|---|---|
| 0 | 青系 | デフォルト |
| 4 | 赤系 | 前年比100%未満の警告 |

---

## 4. 完全なテンプレート

完全なテンプレートは [reference.md](reference.md) を参照。

---

ウィジェットJSON作成時の絶対ルール（format構造、operand形式等）は `/databricks-widget-json-rules` を参照。
