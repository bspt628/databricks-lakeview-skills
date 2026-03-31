---
name: databricks-cli-setup
description: Databricksの使い方ガイド。CLIでのクエリ実行方法と、jupyter-databricks-kernelを使ったローカル開発環境のセットアップ。
---

# Databricks 使い方ガイド

## 重要: MCPツールは使用禁止

**Databricksへのクエリ実行には、MCPツール（`mcp__databricks__*`）を絶対に使用しないこと。**

代わりに、以下の方法を使用する：

1. **databricks CLI** を使った REST API 呼び出し（推奨）
2. **curl** を使った直接 REST API 呼び出し

理由：MCPツールは実行が遅く、不安定な場合がある。REST APIの方が高速で信頼性が高い。

---

## 重要: ローカル版とDatabricks版の使い分け

**スクリプトには必ず2つのバージョンを用意し、環境に応じて厳密に使い分けること。**

### ファイル命名規則

| 環境 | ファイル名 | SQL実行方法 |
|------|-----------|------------|
| **ローカル開発** | `*_local.py` | REST API（databricks CLI / requests） |
| **Databricks本番** | `*_spark.py` | `spark.sql()` |

### 使い分けルール（厳守）

```
# ✅ 正しい使い方
ローカルで開発・テスト → *_local.py を使用
Databricksにアップロード → *_spark.py を使用

# ❌ 絶対に禁止
ローカルで *_spark.py を実行 → sparkオブジェクトがなくエラー、意味がない
Databricksで *_local.py を実行 → REST API経由は遅く、非効率
```

### 現在のファイル構成

```
queries/
├── daily/
│   ├── generate_daily_report_v4_spark.py   # Databricks版（spark.sql使用）
│   └── generate_daily_report_v4_local.py   # ローカル版（REST API使用）
│
└── shop_analysis/
    ├── generate_shop_analysis_spark.py     # Databricks版（spark.sql使用）
    └── generate_shop_analysis_local.py     # ローカル版（REST API使用）
```

### 理由

| 環境 | REST API | spark.sql() |
|------|----------|-------------|
| **ローカル** | ✅ 1〜2秒で高速 | ❌ sparkオブジェクトがない |
| **Databricks** | ❌ HTTP経由で遅い | ✅ 直接実行で最速 |

**この使い分けを守らないと、実行速度が大幅に低下する。**

---

## 1. Databricks CLI の利用方法（推奨）

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

`jupyter-databricks-kernel` を使用して、ローカルからDatabricksクラスタ上でコードを実行する方法を説明します。

### 2.1. 概要

```
ローカルPC (VS Code/Jupyter) → Databricksクラスタ → 結果を返す
```

- **コードはDatabricksクラスタ上で実行**される
- `spark` オブジェクトがそのまま使える
- ローカルのIDEでデバッグ・開発できる

### 2.2. セットアップ

#### インストール

```bash
pip install jupyter-databricks-kernel
python -m jupyter_databricks_kernel.install
```

#### 認証設定

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

---

## 3. 使い方

### 3.1 VS Codeでの使用

1. `.ipynb` ファイルを開く
2. 右上の「カーネルを選択」→「Databricks」を選択
3. セルを実行（クラスタが停止中なら5-6分で起動）

### 3.2 JupyterLabでの使用

```bash
jupyter-lab
```
カーネル選択で「Databricks」を選択

---

## 4. Pythonスクリプトの書き方

### 4.1 Databricks版スクリプトの構造

```python
# Databricks notebook source
# MAGIC %pip install openpyxl

# COMMAND ----------

import pandas as pd
from datetime import date

# spark は自動的に利用可能
def execute_sql(query: str) -> pd.DataFrame:
    return spark.sql(query).toPandas()

# COMMAND ----------

# メイン処理
def main():
    df = execute_sql("SELECT * FROM table LIMIT 10")
    print(df)

main()
```

### 4.2 ローカル版との違い

| 項目 | ローカル版 | Databricks版 |
|------|-----------|--------------|
| SQL実行 | REST API経由 | `spark.sql()` |
| 冒頭 | `#!/usr/bin/env python3` | `# Databricks notebook source` |
| pip | 事前インストール | `# MAGIC %pip install` |
| セル区切り | なし | `# COMMAND ----------` |

### 4.3 変換のポイント

ローカル版 → Databricks版への変換は**3箇所のみ**：

1. **冒頭に追加**
```python
# Databricks notebook source
# MAGIC %pip install openpyxl
# COMMAND ----------
```

2. **SQL実行関数を簡略化**
```python
# Before (ローカル版)
def execute_databricks_sql(query):
    # REST API経由の複雑なコード...

# After (Databricks版)
def execute_databricks_sql(query):
    return spark.sql(query).toPandas()
```

3. **末尾を変更**
```python
# Before
if __name__ == "__main__":
    main()

# After
# COMMAND ----------
main()
```

---

## 5. ファイル同期設定

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

## 6. 出力ファイルの扱い

### 6.1 Workspace保存（推奨）

```python
OUTPUT_DIR = "/Workspace/Users/yourname@company.com/output/"
```

### 6.2 FileStore経由でダウンロード

```python
# クラスタ上で実行
dbutils.fs.cp("file:./output.xlsx", "/FileStore/output.xlsx")
# ブラウザでダウンロード: https://xxx.cloud.databricks.com/files/output.xlsx
```

---

## 7. 注意事項

| 制限 | 詳細 |
|------|------|
| Serverless非対応 | クラシッククラスタが必要 |
| クラスタ起動時間 | 停止中なら5-6分 |
| input()不可 | 対話的入力は使えない |
| ipywidgets不可 | インタラクティブウィジェットは非対応 |

---

## 8. トラブルシューティング

### カーネルが遅い

```toml
# pyproject.toml で不要なファイルを除外
[tool.jupyter-databricks-kernel.sync]
exclude = [".venv/", "__pycache__/", "*.parquet", "data/"]
max_size_mb = 50.0
```

### クラスタが起動しない

```python
# クラスタ状態を確認
from databricks.sdk import WorkspaceClient
w = WorkspaceClient()
cluster = w.clusters.get("your-cluster-id")
print(f"状態: {cluster.state}")
```

---

## 9. 実行例

```python
# Databricks notebook source
# MAGIC %pip install openpyxl

# COMMAND ----------

import pandas as pd
from openpyxl import Workbook

# SQLでデータ取得
df = spark.sql("""
    SELECT store_id, SUM(amount) as total_sales
    FROM your_catalog.your_schema.your_table_name
    WHERE accounting_date >= '2026-01-01'
    GROUP BY store_id
""").toPandas()

print(f"取得件数: {len(df)}")

# COMMAND ----------

# Excelファイル作成
wb = Workbook()
ws = wb.active
ws.title = "売上集計"

# ヘッダー
ws.cell(1, 1, "store_id")
ws.cell(1, 2, "売上合計")

# データ
for i, row in df.iterrows():
    ws.cell(i + 2, 1, row['store_id'])
    ws.cell(i + 2, 2, row['total_sales'])

wb.save("./output.xlsx")
print("Excel出力完了")
```

---

## 10. ディレクトリ構成例

```
queries/daily/
├── generate_daily_report_v2.py           # ローカル版（バックアップ）
└── generate_daily_report_v2_databricks.py # Databricks版（メイン）
```

**推奨**: Databricks版をメインで開発し、ローカル版は互換性のためにバックアップとして保持。
