# テーブルウィジェット テンプレート

## 基本テーブル（書式なし）

```json
{
  "position": { "x": 0, "y": 2, "width": 12, "height": 12 },
  "widget": {
    "name": "sales_table",
    "queries": [
      {
        "name": "main_query",
        "query": {
          "datasetName": "ds_sales",
          "fields": [
            { "name": "store_name", "expression": "`store_name`" },
            { "name": "売上", "expression": "`売上`" },
            { "name": "客数", "expression": "`客数`" }
          ],
          "disaggregated": true
        }
      }
    ],
    "spec": {
      "version": 2,
      "widgetType": "table",
      "frame": {
        "showTitle": true,
        "title": "店舗別売上一覧"
      },
      "encodings": {
        "columns": [
          { "fieldName": "store_name" },
          {
            "fieldName": "売上",
            "format": {
              "type": "number-plain",
              "abbreviation": "none",
              "decimalPlaces": { "type": "exact", "places": 0 },
              "hideGroupSeparator": false
            }
          },
          {
            "fieldName": "客数",
            "format": {
              "type": "number-plain",
              "abbreviation": "none",
              "decimalPlaces": { "type": "exact", "places": 0 },
              "hideGroupSeparator": false
            }
          }
        ]
      }
    }
  }
}
```

## 条件付き書式付きテーブル（前年比赤字表示）

```json
{
  "position": { "x": 0, "y": 2, "width": 12, "height": 15 },
  "widget": {
    "name": "sales_comparison_table",
    "queries": [
      {
        "name": "main_query",
        "query": {
          "datasetName": "ds_sales",
          "fields": [
            { "name": "store_name", "expression": "`store_name`" },
            { "name": "売上", "expression": "`売上`" },
            { "name": "前年売上", "expression": "`前年売上`" },
            { "name": "前年比売上", "expression": "`前年比売上`" }
          ],
          "filters": [
            { "expression": "`row_type` IN ('shop')" }
          ],
          "disaggregated": true
        }
      }
    ],
    "spec": {
      "version": 2,
      "widgetType": "table",
      "frame": {
        "showTitle": true,
        "title": "日次売上実績"
      },
      "encodings": {
        "columns": [
          { "fieldName": "store_name" },
          {
            "fieldName": "売上",
            "format": {
              "type": "number-plain",
              "abbreviation": "none",
              "decimalPlaces": { "type": "exact", "places": 0 },
              "hideGroupSeparator": false
            }
          },
          {
            "fieldName": "前年売上",
            "format": {
              "type": "number-plain",
              "abbreviation": "none",
              "decimalPlaces": { "type": "exact", "places": 0 },
              "hideGroupSeparator": false
            }
          },
          {
            "fieldName": "前年比売上",
            "displayName": "前年比",
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
                    "operand": { "type": "data-value", "value": "1" },
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
        ]
      }
    }
  }
}
```
