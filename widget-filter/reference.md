# フィルターウィジェット 完全テンプレート集

## 1. filter-single-select（最も一般的）

```json
{
  "position": { "x": 0, "y": 0, "width": 4, "height": 2 },
  "widget": {
    "name": "filter_block",
    "queries": [
      {
        "name": "filter_block_query",
        "query": {
          "datasetName": "ds_sales",
          "fields": [
            {
              "name": "region",
              "expression": "`region`"
            }
          ],
          "disaggregated": false
        }
      }
    ],
    "spec": {
      "version": 2,
      "widgetType": "filter-single-select",
      "encodings": {
        "fields": [
          {
            "fieldName": "region",
            "queryName": "filter_block_query"
          }
        ]
      },
      "frame": {
        "showTitle": true,
        "title": "region"
      }
    }
  }
}
```

## 2. filter-single-select（デフォルト値あり）

```json
{
  "position": { "x": 0, "y": 0, "width": 4, "height": 2 },
  "widget": {
    "name": "filter_block",
    "queries": [
      {
        "name": "filter_block_query",
        "query": {
          "datasetName": "ds_sales",
          "fields": [
            {
              "name": "region",
              "expression": "`region`"
            }
          ],
          "disaggregated": false
        }
      }
    ],
    "spec": {
      "version": 2,
      "widgetType": "filter-single-select",
      "encodings": {
        "fields": [
          {
            "fieldName": "region",
            "queryName": "filter_block_query"
          }
        ]
      },
      "frame": {
        "showTitle": true,
        "title": "region"
      },
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
    }
  }
}
```

## 3. filter-multi-select

```json
{
  "position": { "x": 4, "y": 0, "width": 2, "height": 2 },
  "widget": {
    "name": "filter_shops",
    "queries": [
      {
        "name": "filter_shops_query",
        "query": {
          "datasetName": "ds_sales",
          "fields": [
            {
              "name": "store_name",
              "expression": "`store_name`"
            }
          ],
          "disaggregated": false
        }
      }
    ],
    "spec": {
      "version": 2,
      "widgetType": "filter-multi-select",
      "encodings": {
        "fields": [
          {
            "fieldName": "store_name",
            "displayName": "store_name",
            "queryName": "filter_shops_query"
          }
        ]
      },
      "frame": {
        "showTitle": true,
        "title": "store_name"
      }
    }
  }
}
```

## 4. filter-date-picker（デフォルト値あり）

```json
{
  "position": { "x": 8, "y": 0, "width": 4, "height": 2 },
  "widget": {
    "name": "filter_date",
    "queries": [
      {
        "name": "filter_date_query",
        "query": {
          "datasetName": "ds_sales",
          "fields": [
            {
              "name": "accounting_date",
              "expression": "`accounting_date`"
            }
          ],
          "disaggregated": false
        }
      }
    ],
    "spec": {
      "version": 2,
      "widgetType": "filter-date-picker",
      "encodings": {
        "fields": [
          {
            "fieldName": "accounting_date",
            "queryName": "filter_date_query"
          }
        ]
      },
      "frame": {
        "showTitle": true,
        "title": "日付"
      },
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
    }
  }
}
```

## 5. queries[].name の内部ID形式について

UIが自動生成する長い形式のqueryNameには、Databricksが内部的に割り当てるクエリ実行インスタンスIDが含まれる。

```
dashboards/01f1194e61fd.../datasets/01f121e4d2f0..._region
                                     ^^^^^^^^^^^^^^^^
                                     この部分が内部ID
```

内部IDの挙動（実データから判明）:

| ルール | 説明 |
|---|---|
| 同じデータセットでもページごとに内部IDが異なる | `ds_02` が `page_sales` では `01f121e4...` だが `page_group_count` では `01f11d29...` |
| 同じページ内の同じデータセットは同じ内部ID | `page_group_count` の `ds_02` フィルター3つはすべて同一ID |
| UIで編集すると内部IDが変わる | フィルターのデフォルト値変更などでIDが再生成される |

CLI経由で作成する場合は自由な短い名前を使えばこの問題を完全に回避できる。
