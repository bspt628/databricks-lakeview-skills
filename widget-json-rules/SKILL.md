---
name: databricks-widget-json-rules
description: Databricks Lakeview ウィジェットJSONの絶対ルール。テーブル・ピボット・フィルター・カウンターなど、あらゆるウィジェットを新規作成・編集する際に必ず参照する。「ウィジェットを追加して」「新しいテーブルを作って」「グラフを追加して」「ピボットを作って」「ウィジェットのJSONを書いて」など、ウィジェットのJSON構造を生成・編集するすべての作業で使用。
---

# Databricks Lakeview ウィジェット JSON 絶対ルール

このスキルはウィジェット種別（テーブル・ピボット・フィルター・グラフ等）を問わず、
JSON構造を作成・編集するすべての場面で適用される。

---

## 絶対ルール: spec/encodings の構造を独自に考案してはならない

Databricks Lakeview の JSON フォーマットは非公開仕様であり、独自に推測したフォーマットは動作しない。
UIに「視覚化するフィールドを選択してください」と表示されてウィジェットが壊れる。

### よくある間違いフォーマット名（すべて不正）

- `"type": "number"` → 正しくは `"format.type": "number-percent"` 等
- `"numberFormat"` → 正しくは `"format"` オブジェクト
- `"cellStyle"` → 正しくは `"style"`
- `"operator": "LESS_THAN"` → 正しくは `"operator": "<"`
- `"operand.type": "LITERAL"` → 正しくは `"type": "data-value"`
- `"type": "VISUALIZATION_COLOR"` → 正しくは `"themeColorType": "visualizationColors"`

### 正しい方法: 既存の動作しているウィジェットからコピー

必ず同じダッシュボードの動作中の既存ウィジェットから spec 構造をコピーして流用する。

```json
// OK: Databricks Lakeview v2 正規フォーマット
{
  "fieldName": "通常予算比",
  "format": {
    "type": "number-percent",
    "decimalPlaces": { "type": "exact", "places": 1 },
    "hideGroupSeparator": true
  },
  "style": {
    "type": "basic",
    "rules": [{
      "condition": {
        "operand": { "type": "data-value", "value": "1" },
        "operator": "<"
      },
      "foregroundColor": {
        "themeColorType": "visualizationColors",
        "position": 4
      }
    }]
  }
}
```

---

## ウィジェット追加・編集の手順

1. 既存ウィジェットを探す - 同じダッシュボードの同種ウィジェットのJSONを確認する
2. コピーして改変 - 既存ウィジェットをベースにして、必要な差分だけを変更する
3. スキルのテンプレートを使う - 同種ウィジェットのスキルに記載されたテンプレートを使う
   - テーブル: `/databricks-widget-table`
   - ピボット: `/databricks-widget-pivot`
   - フィルター: `/databricks-widget-filter`
   - ダッシュボード全般: `/databricks-dashboard-json`

---

## layoutVersion と グリッド幅12

`/databricks-dashboard-json` スキルの「layoutVersion / GRID_V1」セクションを参照すること。

---

## チェックリスト（ウィジェット追加前に必ず確認）

- [ ] `position.x + position.width <= 12` か（グリッド幅超過は即座にUIが崩れる）
- [ ] `spec.encodings` の構造は既存ウィジェットまたはスキルのテンプレートからコピーしたか
- [ ] `cellStyle` / `numberFormat` / `type: "number"` などの独自フォーマットを使っていないか
- [ ] 条件付き書式の `operator` は `"<"` / `">"` 形式か（`"LESS_THAN"` 等は不正）
- [ ] `operand.type` は `"data-value"` か（`"LITERAL"` は不正）
- [ ] 色指定は `themeColorType: "visualizationColors"` か（`type: "VISUALIZATION_COLOR"` は不正）
- [ ] `queries[].name` はウィジェット種別の正規値か（テーブルは `"main_query"` 必須）
- [ ] データセット削除時に `queries[]` と `encodings.fields[]` の両方から参照を削除したか
- [ ] フィルターが月次+週次の両データセットに連携しているか（`queries` と `encodings.fields` の両方）
