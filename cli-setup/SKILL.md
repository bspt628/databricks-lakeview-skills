---
name: databricks-cli-setup
description: Databricksの使い方ガイド。CLIでのクエリ実行方法と、jupyter-databricks-kernelを使ったローカル開発環境のセットアップ。
---

# Databricks 使い方ガイド

## 1. Databricks CLI の利用方法

### 1.1. warehouse_idについて

- warehouse_id は Serverless SQL Warehouse を探して1つ選択する
- 注意: databricks CLIは設定ファイルの warehouse_id を自動読み込みしないため、毎回JSONに明示的に含める必要がある

### 1.2. 基本的な使い方

```sh
# クエリ実行
databricks api post /api/2.0/sql/statements --profile "DEFAULT" --json '{
  "warehouse_id": "xxxxxxxxxx",
  "catalog": "catalog_name",
  "schema": "schema_name",
  "statement": "select * from table_name limit 10"
}'

# 結果取得（statement_idは実行時に返される値）
databricks api get /api/2.0/sql/statements/{statement_id} --profile "DEFAULT"
```

### 1.3. `databricks` コマンドのコツ

1. クエリ実行のフロー
    - `post`でクエリ実行 → `statement_id`が返る
    - `get`で結果取得（`state`が`SUCCEEDED`になるまで待つ）
    - 長いクエリは`sleep`を挟んで再試行
2. エラー対策
    - `state: CLOSED`: 結果取得が遅すぎた場合。早めに`get`を実行
    - `state: FAILED`: SQLエラー。error_messageを確認
    - `state: RUNNING`: まだ実行中。少し待ってから再度`get`
    - タイムアウト: 大量データの場合は`limit`を付けて確認
3. 結果の読み方
    - `data_array`: 実際のデータ（2次元配列）
    - `schema.columns`: カラム名と型情報
    - `total_row_count`: 総件数（limitかかってても表示）
    - `state`: クエリの実行状態
4. パラメータ付きクエリ

    ```sh
    databricks api post /api/2.0/sql/statements --profile "DEFAULT" --json '{
        "warehouse_id": "xxxxxxxxxx",
        "statement": "select * from table where date >= :start_date",
        "parameters": [{"name": "start_date", "value": "2025-01-01", "type": "DATE"}]
    }'
    ```

### 1.4. curl を使った直接 REST API 呼び出し

databricks CLIが使えない場合、curlで直接APIを叩くことも可能：

```sh
# 環境変数の設定（~/.databrickscfg から取得）
DATABRICKS_HOST="https://dbc-xxxxx.cloud.databricks.com"
DATABRICKS_TOKEN="dapi..."
WAREHOUSE_ID="xxxxxxxxxx"

# クエリ実行
curl -X POST "${DATABRICKS_HOST}/api/2.0/sql/statements" \
  -H "Authorization: Bearer ${DATABRICKS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "warehouse_id": "'"${WAREHOUSE_ID}"'",
    "catalog": "your_catalog",
    "schema": "your_schema",
    "statement": "SELECT * FROM your_table_name LIMIT 10"
  }'

# 結果取得
curl -X GET "${DATABRICKS_HOST}/api/2.0/sql/statements/{statement_id}" \
  -H "Authorization: Bearer ${DATABRICKS_TOKEN}"
```

---

## 2. Databricks Kernel ローカル開発

`jupyter-databricks-kernel` を使用して、ローカルからDatabricksクラスタ上でコードを実行できる。

```
ローカルPC (VS Code/Jupyter) → Databricksクラスタ → 結果を返す
```

- コードはDatabricksクラスタ上で実行される
- `spark` オブジェクトがそのまま使える
- ローカルのIDEでデバッグ・開発できる

### 2.1. セットアップ

```bash
pip install jupyter-databricks-kernel
python -m jupyter_databricks_kernel.install
```

`~/.databrickscfg` にクラスタIDを追加：

```ini
[DEFAULT]
host = https://dbc-xxxxx.cloud.databricks.com/
token = dapi...
cluster_id = 1218-013556-xxxxxx  # ← これを追加
```

クラスタIDは Databricks UI の「Compute」→ クラスタ詳細画面のURLから取得：
```
https://xxx.cloud.databricks.com/#/compute/1218-013556-xxxxxx
                                           ^^^^^^^^^^^^^^^^^^^
```

### 2.2 VS Code / JupyterLabでの使用

- VS Code: `.ipynb` を開き「カーネルを選択」→「Databricks」を選択
- JupyterLab: `jupyter-lab` を起動しカーネル選択で「Databricks」を選択
- クラスタが停止中なら5-6分で起動

---

## 3. Databricks Notebook の書き方

### 3.1 基本構造

```python
# Databricks notebook source
# MAGIC %pip install openpyxl

# COMMAND ----------

import pandas as pd

# spark は自動的に利用可能
def execute_sql(query: str) -> pd.DataFrame:
    return spark.sql(query).toPandas()

# COMMAND ----------

df = execute_sql("SELECT * FROM table LIMIT 10")
print(df)
```

### 3.2 ローカルスクリプトとの主な違い

| 項目 | ローカルスクリプト | Databricks Notebook |
|------|-------------------|---------------------|
| SQL実行 | REST API経由 | `spark.sql()` |
| 冒頭 | `#!/usr/bin/env python3` | `# Databricks notebook source` |
| pip | 事前インストール | `# MAGIC %pip install` |
| セル区切り | なし | `# COMMAND ----------` |

ローカルスクリプト → Notebook への変換ポイント:

1. 冒頭に `# Databricks notebook source` と `# MAGIC %pip install ...` を追加
2. SQL実行を `spark.sql(query).toPandas()` に簡略化
3. `if __name__ == "__main__":` を `# COMMAND ----------` + 直接呼び出しに変更

---

## 4. ファイル同期設定

`pyproject.toml` で同期設定をカスタマイズ：

```toml
[tool.jupyter-databricks-kernel.sync]
enabled = true
source = "."
exclude = ["*.log", "data/", "output/"]
max_size_mb = 100.0
use_gitignore = true
```

---

## 5. 出力ファイルの扱い

### Workspace保存（推奨）

```python
OUTPUT_DIR = "/Workspace/Users/yourname@company.com/output/"
```

### FileStore経由でダウンロード

```python
dbutils.fs.cp("file:./output.xlsx", "/FileStore/output.xlsx")
# ブラウザでダウンロード: https://xxx.cloud.databricks.com/files/output.xlsx
```

---

## 6. 注意事項

| 制限 | 詳細 |
|------|------|
| Serverless非対応 | クラシッククラスタが必要 |
| クラスタ起動時間 | 停止中なら5-6分 |
| input()不可 | 対話的入力は使えない |
| ipywidgets不可 | インタラクティブウィジェットは非対応 |

### トラブルシューティング

カーネルが遅い場合:
```toml
[tool.jupyter-databricks-kernel.sync]
exclude = [".venv/", "__pycache__/", "*.parquet", "data/"]
max_size_mb = 50.0
```

クラスタが起動しない場合:
```python
from databricks.sdk import WorkspaceClient
w = WorkspaceClient()
cluster = w.clusters.get("your-cluster-id")
print(f"状態: {cluster.state}")
```
