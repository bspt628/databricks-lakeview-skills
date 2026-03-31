# 折れ線グラフウィジェット 完全テンプレート集

## 1. 複数系列の折れ線グラフ（前年比推移）

```json
{
  "position": { "x": 0, "y": 2, "width": 12, "height": 12 },
  "widget": {
    "name": "line_sales_yoy",
    "queries": [
      {
        "name": "main_query",
        "query": {
          "datasetName": "ds_timeslot_monthly_pivot",
          "fields": [
            { "name": "年月", "expression": "`年月`" },
            { "name": "avg(morning_sales_yoy)", "expression": "AVG(`morning_sales_yoy`)" },
            { "name": "avg(afternoon_sales_yoy)", "expression": "AVG(`afternoon_sales_yoy`)" },
            { "name": "avg(evening_sales_yoy)", "expression": "AVG(`evening_sales_yoy`)" },
            { "name": "avg(total_sales_yoy)", "expression": "AVG(`total_sales_yoy`)" }
          ],
          "disaggregated": false
        }
      }
    ],
    "spec": {
      "version": 3,
      "widgetType": "line",
      "frame": {
        "showTitle": true,
        "title": "売上前年比推移"
      },
      "encodings": {
        "x": {
          "fieldName": "年月",
          "scale": { "type": "categorical" },
          "axis": { "labelAngle": 45 }
        },
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
      }
    }
  }
}
```

## 2. 売上実績の折れ線グラフ（整数表示）

```json
{
  "position": { "x": 0, "y": 2, "width": 12, "height": 10 },
  "widget": {
    "name": "line_sales_actual",
    "queries": [
      {
        "name": "main_query",
        "query": {
          "datasetName": "ds_timeslot_monthly_pivot",
          "fields": [
            { "name": "年月", "expression": "`年月`" },
            { "name": "sum(morning_sales)", "expression": "SUM(`morning_sales`)" },
            { "name": "sum(afternoon_sales)", "expression": "SUM(`afternoon_sales`)" },
            { "name": "sum(evening_sales)", "expression": "SUM(`evening_sales`)" },
            { "name": "sum(total_sales)", "expression": "SUM(`total_sales`)" }
          ],
          "disaggregated": false
        }
      }
    ],
    "spec": {
      "version": 3,
      "widgetType": "line",
      "frame": {
        "showTitle": true,
        "title": "売上推移"
      },
      "encodings": {
        "x": {
          "fieldName": "年月",
          "scale": { "type": "categorical" }
        },
        "y": {
          "scale": { "type": "quantitative" },
          "fields": [
            { "fieldName": "sum(morning_sales)", "displayName": "morning" },
            { "fieldName": "sum(afternoon_sales)", "displayName": "afternoon" },
            { "fieldName": "sum(evening_sales)", "displayName": "evening" },
            { "fieldName": "sum(total_sales)", "displayName": "合計" }
          ],
          "format": {
            "type": "number-plain",
            "abbreviation": "none",
            "decimalPlaces": { "type": "exact", "places": 0 },
            "hideGroupSeparator": false
          }
        }
      }
    }
  }
}
```

## 3. 線の色・太さ設定（spec.mark）

```json
"mark": {
    "colors": [
        {"themeColorType": "visualizationColors", "position": 2},
        {"themeColorType": "visualizationColors", "position": 7},
        {"themeColorType": "visualizationColors", "position": 1},
        {"themeColorType": "visualizationColors", "position": 4},
        {"themeColorType": "fontColor"},
        {"themeColorType": "visualizationColors", "position": 6},
        {"themeColorType": "visualizationColors", "position": 7},
        {"themeColorType": "visualizationColors", "position": 8},
        {"themeColorType": "visualizationColors", "position": 9},
        {"themeColorType": "visualizationColors", "position": 10}
    ],
    "size": 0.9
}
```

| themeColorType | 説明 |
|---|---|
| `visualizationColors` | テーマのカラーパレットから選択。`position` でインデックス指定 |
| `fontColor` | テーマの文字色（黒/グレー）。基準線に最適 |

### 線の太さ（mark.size）

| size | 見た目 |
|---|---|
| 0.9 | 細め（推奨: 系列が多い場合） |
| 2.0 | 標準（デフォルト） |

## 4. color による系列分割

```json
"color": {
  "fieldName": "時間帯",
  "scale": {
    "type": "categorical",
    "sort": {
      "by": "custom-order",
      "orderedValues": ["morning", "afternoon", "evening", "合計"]
    }
  }
}
```
