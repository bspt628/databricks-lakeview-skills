---
name: databricks-widget-filter
description: Databricks Lakeview ダッシュボードのフィルターウィジェット（filter-single-select, filter-multi-select, filter-date-picker, filter-date-range-picker）のJSON構造リファレンス。フィルターウィジェットの作成・編集時に必ず参照する。「フィルター」「絞り込み」「filter」「ドロップダウン」「日付選択」「date picker」に関するダッシュボード作業で使用。
---

# フィルターウィジェット JSON構造リファレンス

このスキルは、Databricks Lakeview ダッシュボードでフィルターウィジェットを作成するための完全なJSONリファレンス。
本番ダッシュボードの実データから抽出したルールに基づく。

完全なテンプレートは [reference.md](reference.md) を参照。

---

## 1. フィルターの種類（実証済み）

| widgetType | 用途 | 実績数 |
|---|---|---|
| `filter-single-select` | 単一選択ドロップダウン | 53 |
| `filter-multi-select` | 複数選択ドロップダウン | 2 |
| `filter-date-picker` | 日付選択（単一日） | 3 |
| `filter-date-range-picker` | 日付範囲選択 | 未使用（API上は存在） |
| `filter-text-entry` | テキスト入力 | 未使用（API上は存在） |
| `range-slider` | 範囲スライダー | 未使用（API上は存在） |

---

## 2. JSON全体構造

フィルターウィジェットは `pages[].layout[]` の1要素として配置される。

```json
{
  "position": { ... },
  "widget": {
    "name": "<ウィジェット固有名>",
    "queries": [ ... ],
    "spec": { ... }
  }
}
```

---

## 3. 必須要素

### 3.1 position（必須）

```json
{
  "x": 0,        // 0-11（12カラムグリッド）
  "y": 0,        // 上からの行位置
  "width": 4,    // 1-12
  "height": 2    // フィルターは通常2
}
```

- フィルターの典型的なサイズ: `width=2〜4`, `height=2`
- ページ上部（y=0）に横並びで配置するのが一般的

### 3.2 widget.name（必須）

ウィジェットの識別名。自由な文字列でよい。

### 3.3 widget.queries（必須）

```json
"queries": [
  {
    "name": "<クエリ名>",
    "query": {
      "datasetName": "<データセット名>",
      "fields": [ ... ],
      "disaggregated": false
    }
  }
]
```

#### queries[].name（フィルター専用ルール）

フィルターウィジェットでは自由な文字列が使える（データ表示ウィジェットの `main_query` 必須ルールとは異なる）。

CLI経由で作成する場合は短くわかりやすい名前を推奨。UIが自動生成する長いID形式は、UIで編集するたびに変わるため避ける。

`queries[].name` と `encodings.fields[].queryName` は常に完全一致させること。

#### query.fields（必須）

最低1つのフィールド（フィルター対象カラム）が必要。

基本形:
```json
"fields": [
  {
    "name": "store_name",
    "expression": "`store_name`"
  }
]
```

`associativity` フィールド（UIが自動生成）はなくても動作する。CLI経由で手作りする場合は省略してよい。

#### query.disaggregated（必須）

フィルターでは常に `false`。テーブル・ピボットの `true` とは逆なので注意。

### 3.4 spec（必須）

```json
"spec": {
  "version": 2,
  "widgetType": "filter-single-select",
  "encodings": {
    "fields": [
      {
        "fieldName": "region",
        "queryName": "<queries[].nameと同じ値>"
      }
    ]
  },
  "frame": {
    "showTitle": true,
    "title": "region"
  }
}
```

- `spec.version`: フィルターは常に `2`
- `fieldName`: フィルター対象のカラム名
- `queryName`: `widget.queries[].name` と完全一致させる
- `displayName`（オプション）: 表示名を変更する場合に使用

---

## 4. オプション要素

### 4.1 selection（デフォルト値の設定）

`spec` 直下に配置する（`encodings` と同階層）。

STRING型:
```json
"selection": {
  "defaultSelection": {
    "values": {
      "dataType": "STRING",
      "values": [
        { "value": "Region A" }
      ]
    }
  }
}
```

DATE型:
```json
"selection": {
  "defaultSelection": {
    "values": {
      "dataType": "DATE",
      "values": [
        { "value": "now-1d/d" }
      ]
    }
  }
}
```

相対日付の書式: `"now-1d/d"`(昨日), `"now/d"`(今日), `"now-7d/d"`(7日前)

### 4.2 encodings.fields[].displayName

フィルターのラベルをカラム名と異なる表示にしたい場合:

```json
{
  "fieldName": "store_name",
  "displayName": "店舗",
  "queryName": "filter_shop_query"
}
```

---

## 5. フィルターの連動ルール

フィルターは同一ページ内の同じ `datasetName` を持つウィジェットに自動で適用される。
異なるデータセットには影響しない。

---

## 6. データセット削除時の注意（最重要）

データセットを削除する際は、フィルターウィジェットの以下の2箇所を両方確認して参照を削除すること。

1. `widget.queries[]` — 該当datasetNameを参照するクエリオブジェクトを削除
2. `widget.spec.encodings.fields[]` — 削除したクエリのqueryNameを参照するエントリを削除

チェック方法（Python）:
```python
for page in dash['pages']:
    for li in page['layout']:
        w = li['widget']
        if 'filter' not in w.get('spec', {}).get('widgetType', ''):
            continue
        query_names = {q['name'] for q in w.get('queries', [])}
        enc_names = {f['queryName'] for f in w['spec']['encodings']['fields']}
        orphaned = enc_names - query_names
        if orphaned:
            print(f"ERROR: {w['name']} に孤立したqueryName参照: {orphaned}")
```

---

ウィジェットJSON作成時の絶対ルール（format構造、operand形式等）は `/databricks-widget-json-rules` を参照。
