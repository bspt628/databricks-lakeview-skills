# Databricks Lakeview Dashboard Skills for Claude Code

Databricks Lakeview (AI/BI Dashboards) のダッシュボードJSONを手書きするための Claude Code スキル集。

Lakeview の `serialized_dashboard` 内部JSON構造は公式ドキュメントに記載されていない。このスキル集は、本番ダッシュボード（9ダッシュボード、137ウィジェット）のリバースエンジニアリングにより抽出したルールに基づいている。

## なぜ必要か

Lakeview のUIエディタは基本的な操作には十分だが、以下のケースではJSONの直接編集が必要になる:

- CI/CDパイプラインでのダッシュボード管理（Databricks Asset Bundles, Terraform）
- UIでは設定できない細かい書式設定（条件付き書式の閾値、カラースケールのdomain指定）
- ダッシュボードのテンプレート化・複製
- `databricks lakeview create/update` CLIによるプログラマティックな操作

公式APIドキュメントは外側のエンベロープ（`dashboard_id`, `display_name`, `warehouse_id`）のみをカバーしており、`serialized_dashboard` 内部の `pages`, `layout`, `widget`, `spec`, `encodings` の構造は非公開仕様。このスキル集がそのギャップを埋める。

## スキル一覧

| スキル | 内容 |
|-------|------|
| `dashboard-json` | ダッシュボードJSON全体構造、CLI操作、データセット管理、パラメータ設定 |
| `widget-table` | テーブルウィジェットのJSON構造、カラム書式、条件付き書式 |
| `widget-pivot` | ピボットテーブルのJSON構造、cubeGroupingSets、カラースケール |
| `widget-line` | 折れ線グラフのJSON構造、複数系列、Y軸フォーマット |
| `widget-filter` | フィルターウィジェット（single-select, multi-select, date-picker） |
| `widget-json-rules` | ウィジェットJSON作成時の絶対ルール（独自フォーマット禁止） |
| `large-json-safety` | 大規模JSON（1MB+）のクラッシュ防止ガイド |
| `cli-setup` | Databricks CLIの使い方、jupyter-databricks-kernelセットアップ |
| `sql-reference` | Databricks固有のSQL構文リファレンス |

## インストール

Claude Codeプロジェクトの `.claude/skills/` 配下にcloneまたはsubmodule追加:

```bash
# submoduleとして追加（推奨）
git submodule add https://github.com/bspt628/databricks-lakeview-skills.git .claude/skills/databricks

# または直接clone
git clone https://github.com/bspt628/databricks-lakeview-skills.git .claude/skills/databricks
```

Claude Codeがディレクトリ内のSKILL.mdを自動検出し、関連する作業時にスキルがトリガーされる。

## 注意: spec.version の重要性

Lakeviewウィジェットは `spec.version` の値がウィジェットタイプごとに固定されており、間違った値を指定するとウィジェットが表示されない。この仕様は公式ドキュメントに記載されていない。

| spec.version | 対象ウィジェット |
|-------------|----------------|
| 2 | table, counter, filter-* |
| 3 | pivot, line, bar, combo, area, pie, heatmap |

同様に、`queries[].name` はデータ表示ウィジェットでは `"main_query"` でなければデータが表示されない（フィルターウィジェットは自由な文字列でよい）。このような「設定しないと動かないが、ドキュメントに書かれていない」仕様を体系的にまとめたのがこのスキル集の価値。

## 免責事項

- このスキル集は Databricks Lakeview の非公開内部仕様のリバースエンジニアリングに基づいている
- 2026年3月時点の情報であり、Databricksのアップデートにより構造が変更される可能性がある
- AWS/Azure/GCP全てのDatabricksワークスペースで同一の構造が確認されているが、将来の互換性は保証されない
- Databricksの公式サポート対象外の手法を含む

## ライセンス

MIT
