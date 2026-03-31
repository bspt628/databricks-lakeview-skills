---
name: databricks-dashboard-json
description: DatabricksダッシュボードのJSON全体構造、CLI操作、運用ルール、データセット管理、パラメータ設定。ダッシュボードJSON、lvdash.json、CLI操作に関する質問や作業時に使用。ウィジェット個別の構造は各widget-*スキルを参照。
---

# Databricks Lakeview ダッシュボード JSON構造

実際のダッシュボード9つを調査・テストして実証済みのルールと知見。
ウィジェット個別の構造は `/databricks-widget-table`, `/databricks-widget-pivot`, `/databricks-widget-line`, `/databricks-widget-filter` を参照。
ウィジェットJSON作成時の絶対ルールは `/databricks-widget-json-rules` を参照。

---

## 1. JSON全体構造

```
{
  "dashboard_id": "01f1194e...",          // Databricksが自動付与（createでは不要）
  "display_name": "ダッシュボード名",
  "warehouse_id": "<YOUR_WAREHOUSE_ID>",
  "serialized_dashboard": "{ ... }",      // API上はJSON文字列（エスケープ済み）
  ...
}
```

`serialized_dashboard` の中身:

```
{
  "datasets": [...],    // SQLデータソース定義
  "pages": [...],       // ページ（タブ）定義。各ページにwidgetのlayoutを持つ
  "uiSettings": {...}   // UI設定
}
```

### ローカル保存方針

- `serialized_dashboard` はパース済みオブジェクトとして保存（可読性のため）
- API送信時にPythonで `json.dumps()` して文字列に戻す
- ファイル配置: ローカルに保存する場合は任意のディレクトリにdashboard.jsonとして保存

---

## 2. CLI操作

```bash
# 一覧
databricks lakeview list --output json

# 取得
databricks lakeview get <DASHBOARD_ID> --output json

# 作成（新規）
databricks lakeview create --json @payload.json --output json

# 更新（既存を編集）
databricks lakeview update <DASHBOARD_ID> --json @payload.json --output json

# 削除（ゴミ箱へ。`delete`コマンドは存在しない）
databricks lakeview trash <DASHBOARD_ID>

# 公開
databricks lakeview publish <DASHBOARD_ID>

# 公開済み取得
databricks lakeview get-published <DASHBOARD_ID>
```

### create/update 用 payload の作り方

```python
import json, subprocess

with open("path/to/dashboard.json") as f:
    data = json.load(f)

payload = {
    "display_name": data["display_name"],
    "warehouse_id": data["warehouse_id"],
    "serialized_dashboard": json.dumps(data["serialized_dashboard"], ensure_ascii=False)
}

with open("/tmp/payload.json", "w") as f:
    json.dump(payload, f, ensure_ascii=False)

# create
subprocess.run(["databricks", "lakeview", "create", "--json", "@/tmp/payload.json", "--output", "json"])

# update
subprocess.run(["databricks", "lakeview", "update", DASHBOARD_ID, "--json", "@/tmp/payload.json", "--output", "json"])
```

### 安全なテスト手順

1. ローカルJSONを編集
2. 既存を更新せず、別名で `create` して動作確認
3. 問題なければ本番に `update`
4. テスト用は `trash` で削除

---

## 3. ダッシュボード変更の運用ルール

### 変更フロー（必ずこの順序で実行）

```
1. 【必須】`databricks lakeview get` でリモートの最新状態を取得
2. ローカルの dashboard.json と取得結果の差分確認
3. ローカルで最小限の差分のみ変更
4. テスト用ダッシュボードで動作確認（create → 確認 → trash）
5. 問題なければ本番に update
6. ローカルのJSONとSQLを更新してgitコミット
```

### CLI操作前のリモート同期（最重要ルール）

`databricks lakeview update` を実行する前に、必ず `databricks lakeview get` でリモートの最新状態を取得すること。

ローカルの古いJSONで `update` すると、UI上の変更が全て消える。

### 禁止事項

- リモート同期なしで `databricks lakeview update` を実行しない
- ダッシュボードJSONを丸ごと上書きしない

### データセット削除時の必須手順

データセットを`datasets`配列から削除する際は、以下の3箇所すべてから参照を削除すること:

1. `datasets[]` — データセット本体を削除
2. 全ウィジェットの `queries[]` — 該当datasetNameを参照するクエリを削除
3. フィルターウィジェットの `spec.encodings.fields[]` — 削除したクエリのqueryNameを参照するエントリを削除

検証スクリプト:
```python
ds_names = {ds['name'] for ds in dash['datasets']}
for page in dash['pages']:
    for li in page['layout']:
        w = li['widget']
        for q in w.get('queries', []):
            if q['query']['datasetName'] not in ds_names:
                print(f"ERROR: {w['name']} が存在しないデータセット {q['query']['datasetName']} を参照")
        if 'filter' in w.get('spec', {}).get('widgetType', ''):
            query_names = {q['name'] for q in w.get('queries', [])}
            for f in w['spec']['encodings']['fields']:
                if f['queryName'] not in query_names:
                    print(f"ERROR: {w['name']} の encodings に孤立したqueryName {f['queryName']}")
```

---

## 4. queries[].name のルール（実証済み）

### データ表示ウィジェット → `main_query` 必須

table, pivot, bar, line, counter, pie, combo, area, heatmap 等のすべてで `main_query` が100%使われている。カスタム文字列を設定するとデータが表示されない。

### フィルターウィジェット → 自由な文字列でOK

CLI経由で作成する場合は短くわかりやすい名前を推奨。

---

## 5. グリッドレイアウト

- 12列グリッド: `x`: 0〜11, `width`: 1〜12
- `y`: 0以上の整数（下方向に無制限）

```json
// 全幅: {"x": 0, "y": 0, "width": 12, "height": 15}
// 2列: {"x": 0, "width": 6}, {"x": 6, "width": 6}
// 3列: {"x": 0, "width": 4}, {"x": 4, "width": 4}, {"x": 8, "width": 4}
// フィルター: {"width": 2, "height": 1} or {"width": 4, "height": 2}
```

### layoutVersion（必須）

全ページに `"layoutVersion": "GRID_V1"` を設定すること。未設定のページは旧仕様の24列グリッドとして扱われ、widthが2倍に拡大される。

---

## 6. データセット（datasets）

### 構造

```json
{
  "name": "ds_01_member_composition",
  "displayName": "01_会員区分",
  "queryLines": [
    "SELECT\n",
    "    store_id AS `store_id`,\n",
    "    store_name AS `store_name`\n",
    "FROM table_name\n"
  ],
  "parameters": [...]  // オプション: 日付パラメータ等
}
```

### queryLines の書式ルール

全ての行は末尾に `\n` を持つ。空行は `"\n"` として含める。

```python
# SQLファイル → queryLines
def sql_file_to_query_lines(file_path: str) -> list[str]:
    with open(file_path) as f:
        sql = f.read()
    return [line + '\n' for line in sql.split('\n')]

# queryLines → SQL
sql = ''.join(dataset['queryLines'])
```

### SQLカラム名とウィジェットの紐付け

SQLの `AS \`カラム名\`` がウィジェットの `fields[].expression` でバッククォートで参照される。日本語カラム名推奨。

### データセット追加時のチェックリスト

- [ ] `name` がダッシュボード内で一意か
- [ ] `displayName` が設定されているか
- [ ] `queryLines` の全行が `\n` で終わっているか
- [ ] SQLが単体で実行可能か
- [ ] ウィジェットの `datasetName` と `name` が完全一致しているか


---

## 7. 日付パラメータウィジェット

### データセットのパラメータ定義

```json
{
  "name": "ds_period_summary",
  "queryLines": [...],
  "parameters": [
    {
      "displayName": "param_start_date",
      "keyword": "param_start_date",
      "dataType": "DATE",
      "defaultSelection": {
        "values": {"dataType": "DATE", "values": [{"value": "now/M"}]}
      }
    }
  ]
}
```

デフォルト値: `now/M`(当月初日), `now-1d/d`(昨日), `now/d`(今日)

SQL内での参照: `WHERE target_date >= :param_start_date`

### 日付ピッカーウィジェット

```json
{
  "widget": {
    "name": "start_date_picker",
    "queries": [
      {
        "name": "param_start_query",
        "query": {
          "datasetName": "ds_period_summary",
          "parameters": [{"name": "param_start_date", "keyword": "param_start_date"}],
          "disaggregated": false
        }
      }
    ],
    "spec": {
      "version": 2,
      "widgetType": "filter-date-picker",
      "encodings": {
        "fields": [
          {"parameterName": "param_start_date", "queryName": "param_start_query"}
        ]
      },
      "frame": {"showTitle": true, "title": "開始日"}
    }
  }
}
```

パラメータウィジェットでは `fieldName` ではなく `parameterName` を使用。

---

## 8. テキストウィジェット

```json
{
  "widget": {
    "name": "text_note",
    "multilineTextboxSpec": {
      "lines": ["テキスト内容\n", "\n", "[リンク](https://example.com)"]
    }
  },
  "position": {"x": 0, "y": 0, "width": 12, "height": 2}
}
```

---

## 9. ウィジェット固有の設定（相互参照）

以下の設定は各ウィジェットスキルを参照すること:

- 数値フォーマット（カンマ区切り、パーセント表示）→ `/databricks-widget-table`, `/databricks-widget-pivot`
- 条件付き書式（文字色ルール、カラースケール）→ `/databricks-widget-table`, `/databricks-widget-pivot`
- ソート順（年代・時間帯の custom-order）→ `/databricks-widget-table`, `/databricks-widget-pivot`
- spec.version と widgetType の対応 → `/databricks-widget-json-rules`
