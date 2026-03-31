---
name: databricks-sql-reference
description: Databricks SQLの文法リファレンス。型変換、正規表現、文字列操作、日付関数、配列操作、パフォーマンス最適化の手法。SQLクエリ作成時に参照。
---

# Databricks SQL リファレンス

標準SQLの知識は前提とし、Databricks固有の関数・制約を記載する。

---

## 1. Databricks固有の関数

### 安全な型変換（変換失敗時にエラーではなくNULLを返す）
```sql
TRY_CAST(column AS INT)
TRY_CAST(column AS DATE)
TRY_CAST(column AS BIGINT)
```

### 正規表現関数
```sql
-- パターンマッチング（true/false）
REGEXP_LIKE(column, '^[0-9]+:')

-- パターン抽出（グループ番号で指定）
REGEXP_EXTRACT(column, '^([0-9]+):', 1)

-- パターン置換
REGEXP_REPLACE(column, '（.*）', '')
```

### 連番配列生成
```sql
-- SEQUENCE + EXPLODEで連番行を生成（例: 0から27までの28行）
EXPLODE(SEQUENCE(0, 27))
```

---

## 2. 権限制約

### 使用に注意
- `CREATE TABLE` - ワークスペースの権限設定によっては使用不可の場合がある
- `CREATE MATERIALIZED VIEW` - ワークスペースの権限設定によっては使用不可の場合がある

### 使用可能（ただし基本はCTEで十分）
```sql
-- セッションスコープの一時ビュー（セッション終了時に削除）
CREATE OR REPLACE TEMP VIEW view_name AS
SELECT ...;

-- メモリキャッシュ（大量データには不向き）
CACHE TABLE temp_table_name AS
SELECT ...;
```

---

## 3. 識別子のバッククォートルール

### 日本語カラム名には必ずバッククォートを付ける

```sql
-- エラーになる
SELECT
    COALESCE(m.region, 'N/A') AS region,
    SUM(sales) AS 売上

-- 正しい書き方
SELECT
    COALESCE(m.region, 'N/A') AS `region`,
    SUM(sales) AS `売上`
```

### バッククォートが必須な場合
- 日本語を含む: `売上`, `store_name`, `年月`
- ハイフンを含む: `shop-code`
- スペースを含む: `shop name`
- 数字で始まる: `1st_column`
- 予約語: `select`, `from`, `table`

### バッククォート不要な場合
ASCII英数字とアンダースコアのみで、数字で始まらない場合:
```sql
SELECT store_id, sales_amount, year_month_01
```

### 日本語カラム名の使用例
```sql
SELECT
    DATE_FORMAT(t.accounting_date, 'yyyy-MM') AS `年月`,
    COALESCE(m.region, 'N/A') AS `region`,
    COALESCE(m.area, 'N/A') AS `area`,
    COALESCE(m.store_name, 'Unknown') AS `store_name`,
    t.store_id AS `store_id`,
    SUM(t.sales_wo_tax) AS `売上`,
    COUNT(*) AS `組数`,
    SUM(t.number_of_people) AS `客数`,
    ROUND(SUM(t.sales_wo_tax) / NULLIF(SUM(t.number_of_people), 0), 0) AS `客単価`
FROM ...
```

---

## 4. 最適化ガイドライン

### 早期フィルタリング
```sql
-- 最初にWHERE句で絞り込む
WHERE accounting_date >= ADD_MONTHS(CURRENT_DATE(), -24)
  AND store_id IN (SELECT store_id FROM active_stores)
```

### JOIN前の絞り込み
```sql
-- 事前に絞り込んでからJOIN
WITH active_members AS (
    SELECT DISTINCT member_id FROM transactions
    WHERE accounting_date >= '2024-01-01'
)
SELECT ... FROM members m
INNER JOIN active_members am ON m.member_id = am.member_id
```

### CTEで段階的に分割
```sql
WITH
step1 AS (SELECT ... FROM table1 WHERE ...),
step2 AS (SELECT ... FROM step1 JOIN table2 ...),
final AS (SELECT ... FROM step2)
SELECT * FROM final
```

### 避けるべきこと
- テーブル作成（権限が必要）
- 中間テーブルの乱立
- 過度に複雑な最適化構造
