---
name: databricks-large-json-safety
description: Databricksダッシュボードの大きなJSON（1MB以上）を扱う際のクラッシュ防止ガイド。ダッシュボードの作成・更新・コピー時に必ず参照。セッションクラッシュ、メモリ不足、コマンドライン制限を回避する方法。
---

# Databricks 大規模JSON処理ガイド（クラッシュ防止）

## クラッシュの原因

| 原因 | 症状 |
|------|------|
| コマンドライン引数制限 | `argument list too long` エラー |
| メモリ不足 | セッションが応答しなくなる |
| 大きなファイルの直接読み込み | `File content exceeds maximum` エラー |
| Bashツールでの大量出力 | タイムアウトまたはクラッシュ |

---

## 必須ルール

### 1. 大きなJSONは絶対にBashの引数に渡さない

```bash
# NG
databricks lakeview create --serialized-dashboard "$(cat large.json)"

# OK: --json @ファイルパス を使う
databricks lakeview update ID --json @/tmp/request.json --profile DEFAULT
```

### 2. JSONの処理はPythonで行う

以下のテンプレートを使い、取得・変更・保存・CLI実行をすべてPython内で完結させる。

```python
python3 << 'PYEOF'
import json
import subprocess

# 1. 既存ダッシュボードを取得
result = subprocess.run(
    ['databricks', 'lakeview', 'get', 'DASHBOARD_ID', '--profile', 'DEFAULT'],
    capture_output=True, text=True
)
dashboard_info = json.loads(result.stdout)
dashboard = json.loads(dashboard_info['serialized_dashboard'])

# 2. 変更を加える（例: データセット追加）
new_dataset = {
    "name": "new_data",
    "displayName": "新しいデータセット",
    "queryLines": ["SELECT * FROM table\n"]
}
dashboard['datasets'].append(new_dataset)

# 3. リクエストを作成してファイル保存
request = {
    "display_name": dashboard_info.get('display_name', 'Dashboard'),
    "serialized_dashboard": json.dumps(dashboard, ensure_ascii=False)
}
with open('/tmp/update_request.json', 'w') as f:
    json.dump(request, f, ensure_ascii=False)

# 4. CLIで更新
result = subprocess.run(
    ['databricks', 'lakeview', 'update', 'DASHBOARD_ID',
     '--json', '@/tmp/update_request.json', '--profile', 'DEFAULT'],
    capture_output=True, text=True, timeout=60
)

if result.returncode == 0:
    print("更新成功!")
else:
    print(f"エラー: {result.stderr[:500]}")
PYEOF
```

### 3. 大きなファイルの読み取りを避ける

```bash
# NG: 大きな出力を直接読まない
cat /tmp/large_output.txt

# OK: tail/head で一部だけ確認
tail -20 /tmp/large_output.txt

# OK: jq で必要な部分だけ抽出
jq '.datasets | length' /tmp/dashboard.json
jq '.pages[].displayName' /tmp/dashboard.json
```

### 4. ダッシュボード情報の確認は軽量コマンドで

```bash
# データセット数
databricks lakeview get ID --profile DEFAULT | jq -r '.serialized_dashboard' | jq '.datasets | length'

# ページ名一覧
databricks lakeview get ID --profile DEFAULT | jq -r '.serialized_dashboard' | jq '.pages[].displayName'
```

---

## チェックリスト

ダッシュボード操作前に確認:

- [ ] JSONサイズが大きい場合（>100KB）はPython処理を使用
- [ ] `--json @filepath` 形式でCLI実行
- [ ] 大きなファイルは直接読み込まない（tail/jqを使用）
- [ ] subprocess.run に timeout を設定
- [ ] 出力確認は軽量コマンドで
