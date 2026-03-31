# ピボットテーブル 完全テンプレート集

## 1. 基本ピボット（日付×店舗 の予算表）

```json
{
  "position": { "x": 0, "y": 2, "width": 12, "height": 18 },
  "widget": {
    "name": "pivot_budget",
    "queries": [
      {
        "name": "main_query",
        "query": {
          "datasetName": "ds_budget",
          "fields": [
            { "name": "日付", "expression": "`日付`" },
            { "name": "region", "expression": "`region`" },
            { "name": "store_name", "expression": "`store_name`" },
            { "name": "通常予算", "expression": "`通常予算`" },
            { "name": "目標予算", "expression": "`目標予算`" }
          ],
          "filters": [
            { "expression": "`row_type` IN ('shop')" }
          ],
          "cubeGroupingSets": {
            "sets": [
              { "fieldNames": ["日付"] },
              { "fieldNames": ["region", "store_name"] }
            ]
          },
          "disaggregated": true,
          "orders": [
            { "direction": "ASC", "expression": "`日付`" },
            { "direction": "ASC", "expression": "`region`" },
            { "direction": "ASC", "expression": "`store_name`" }
          ]
        }
      }
    ],
    "spec": {
      "version": 3,
      "widgetType": "pivot",
      "frame": {
        "showTitle": true,
        "title": "日次予算表"
      },
      "encodings": {
        "rows": [
          { "fieldName": "日付" }
        ],
        "columns": [
          { "fieldName": "region" },
          { "fieldName": "store_name", "headerHeight": 90 }
        ],
        "cell": {
          "type": "multi-cell",
          "fields": [
            {
              "fieldName": "通常予算",
              "cellType": "text",
              "format": {
                "type": "number-plain",
                "abbreviation": "none",
                "decimalPlaces": { "type": "exact", "places": 0 },
                "hideGroupSeparator": false
              }
            },
            {
              "fieldName": "目標予算",
              "cellType": "text",
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
}
```

## 2. カスタムソート + カラースケール付きピボット

```json
{
  "position": { "x": 0, "y": 2, "width": 12, "height": 20 },
  "widget": {
    "name": "pivot_age_timeslot_sales",
    "queries": [
      {
        "name": "main_query",
        "query": {
          "datasetName": "ds_age_timeslot",
          "fields": [
            { "name": "region", "expression": "`region`" },
            { "name": "area", "expression": "`area`" },
            { "name": "時間帯", "expression": "`時間帯`" },
            { "name": "年月", "expression": "`年月`" },
            { "name": "年代", "expression": "`年代`" },
            { "name": "売上", "expression": "`売上`" },
            { "name": "売上前年比", "expression": "`売上前年比`" }
          ],
          "cubeGroupingSets": {
            "sets": [
              { "fieldNames": ["region", "area", "時間帯"] },
              { "fieldNames": ["年月", "年代"] }
            ]
          },
          "disaggregated": true,
          "orders": [
            { "direction": "ASC", "expression": "`region`" },
            { "direction": "ASC", "expression": "`area`" },
            { "direction": "ASC", "expression": "`年月`" }
          ]
        }
      }
    ],
    "spec": {
      "version": 3,
      "widgetType": "pivot",
      "frame": {
        "showTitle": true,
        "title": "年代別×時間帯別 売上前年比"
      },
      "encodings": {
        "rows": [
          { "fieldName": "region", "displayName": "region" },
          { "fieldName": "area", "displayName": "area" },
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
        ],
        "columns": [
          { "fieldName": "年月", "displayName": "年月" },
          {
            "fieldName": "年代",
            "scale": {
              "type": "categorical",
              "sort": {
                "by": "custom-order",
                "orderedValues": [
                  "中高生以下", "大学生", "20代", "30代",
                  "40代", "50代", "60代以上",
                  "生年月日不明", "非会員", "全年代"
                ]
              }
            },
            "displayName": "年代"
          }
        ],
        "cell": {
          "type": "multi-cell",
          "fields": [
            {
              "fieldName": "売上",
              "cellType": "text",
              "format": {
                "type": "number-plain",
                "abbreviation": "none",
                "decimalPlaces": { "type": "exact", "places": 0 },
                "hideGroupSeparator": false
              }
            },
            {
              "fieldName": "売上前年比",
              "cellType": "text",
              "format": {
                "type": "number-percent",
                "decimalPlaces": { "type": "exact", "places": 1 },
                "hideGroupSeparator": true
              },
              "style": {
                "type": "color-scale",
                "backgroundColor": {
                  "scale": {
                    "type": "quantitative",
                    "colorRamp": {
                      "mode": "scheme",
                      "scheme": "redblue"
                    },
                    "domain": {
                      "mid": 1,
                      "max": 2,
                      "min": 0
                    }
                  }
                }
              }
            }
          ]
        }
      }
    }
  }
}
```

## 3. 前年比カラースケールのスタイル（コピー用）

```json
{
  "style": {
    "type": "color-scale",
    "backgroundColor": {
      "scale": {
        "type": "quantitative",
        "colorRamp": {
          "mode": "scheme",
          "scheme": "redblue"
        },
        "domain": {
          "mid": 1,
          "max": 2,
          "min": 0
        }
      }
    }
  }
}
```
