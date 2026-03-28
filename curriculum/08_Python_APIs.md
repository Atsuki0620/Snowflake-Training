# Chapter 8：Snowflake Python APIs でオブジェクト管理

**目安時間**: 60〜90分
**ゴール**: PythonコードからSnowflakeのオブジェクト（テーブル・タスクなど）を管理できる

← [Chapter 7](07_Taskによる定期自動実行.md) | [目次](README.md)

---

## このChapterで何をするか

これまでSQLでやっていた「テーブル作成・タスクの管理」などを
Pythonのオブジェクト操作として扱う方法を学びます。

> 💡 **Python APIs と Snowflake Connector の違い**
>
> | | Snowflake Connector | Snowflake Python APIs |
> |---|---|---|
> | 操作方法 | SQLを文字列で渡して実行 | Pythonオブジェクトとして操作 |
> | 主な用途 | データ読み書き・PUTコマンド | オブジェクト管理（DB・テーブル・タスク） |
> | 例 | `cur.execute("SELECT ...")` | `root.databases["X"].schemas.iter()` |
>
> **やらないと？** オブジェクト管理のたびにSQL文字列を組み立てる必要があり、
> タイポや動的SQL生成のバグが起きやすくなります。
> **やると？** Pythonらしい直感的なコードで管理でき、IDEの補完も効きます。

---

## 8-1 Python APIs の初期化

```python
# 必要なクラスをインポート
from snowflake.core import Root            # APIs のエントリーポイント
from snowflake.core.database import Database
from snowflake.core.schema import Schema
from snowflake.core.table import Table, TableColumn
from snowflake.core.task import Task, StoredProcedureCall
import snowflake.connector

# まず通常のコネクターで接続を確立
conn = snowflake.connector.connect(
    account="<アカウント識別子>",  # 例: abc12345.ap-northeast-1.aws
    user="<ユーザー名>",
    password="<パスワード>",
    warehouse="COMPUTE_WH",        # Taskの操作などにウェアハウスが必要
    database="SENSOR_LAB",         # デフォルトDB
    schema="BRONZE"                # デフォルトスキーマ
)

# Root オブジェクトを作成（これが Python APIs の起点）
# root から辿ることで全てのSnowflakeオブジェクトにアクセスできる
root = Root(conn)
```

---

## 8-2 データベース・スキーマの操作

> 💡 **`iter()` で一覧を取得する方法**
> Snowflake Python APIsでは、オブジェクトのコレクション（`databases`・`schemas` など）に
> `.iter()` を呼ぶことで全件を取得できます。
> SQLの `SHOW DATABASES` に相当します。

```python
# データベース一覧取得（SHOW DATABASES 相当）
databases = root.databases.iter()  # イテレータを返す
for db in databases:
    print(db.name)  # データベース名を出力

# 特定DBのスキーマ一覧取得（SHOW SCHEMAS IN DATABASE SENSOR_LAB 相当）
schemas = root.databases["SENSOR_LAB"].schemas.iter()
for s in schemas:
    print(s.name)

# 新しいスキーマをPythonオブジェクトとして作成
new_schema = Schema(name="ARCHIVE")  # スキーマオブジェクトを定義
root.databases["SENSOR_LAB"].schemas.create(
    new_schema,
    mode="if_not_exists"  # 既に存在する場合はスキップ（CREATE IF NOT EXISTS 相当）
)
print("スキーマ ARCHIVE を作成しました")
```

---

## 8-3 テーブルのオブジェクト管理

> 💡 **`fetch()` でテーブルの詳細を取得する**
> `.fetch()` はSnowflakeからそのオブジェクトの最新情報を取得します。
> カラム定義・コメント・オプションなど、`DESCRIBE TABLE` で得られる情報が取れます。

```python
# テーブル一覧取得（SHOW TABLES IN SCHEMA SENSOR_LAB.BRONZE 相当）
tables = root.databases["SENSOR_LAB"].schemas["BRONZE"].tables.iter()
for t in tables:
    print(f"テーブル: {t.name}")

# テーブルの詳細取得（DESCRIBE TABLE RAW_SENSOR 相当）
table = root.databases["SENSOR_LAB"].schemas["BRONZE"].tables["RAW_SENSOR"].fetch()
print(f"テーブル名: {table.name}")
print(f"カラム数: {len(table.columns)}")
for col in table.columns:
    print(f"  {col.name}: {col.datatype}")  # 各カラムの名前と型を表示
```

---

## 8-4 Task の Python API 管理

> 💡 **PythonからTaskを操作するメリット**
> Snowsightで手動操作するよりも、Pythonスクリプトに組み込んで
> 「デプロイ後に自動でTaskをResumeする」などの運用自動化ができます。

```python
from snowflake.core.task import Task

# タスク一覧取得（SHOW TASKS 相当）
tasks = root.databases["SENSOR_LAB"].schemas["BRONZE"].tasks.iter()
for t in tasks:
    print(f"タスク: {t.name}, 状態: {t.state}")  # state: started / suspended

# タスクへの参照を取得（実際の操作はこのオブジェクト経由で行う）
task_ref = root.databases["SENSOR_LAB"].schemas["BRONZE"].tasks["TASK_BRONZE"]

# タスクを RESUME（有効化）: ALTER TASK ... RESUME 相当
task_ref.resume()
print("TASK_BRONZE を有効化しました")

# タスクを SUSPEND（停止）: ALTER TASK ... SUSPEND 相当
task_ref.suspend()
print("TASK_BRONZE を停止しました")

# タスクの詳細情報取得（スケジュール設定・SQL定義などが取れる）
task_detail = task_ref.fetch()
print(f"スケジュール: {task_detail.schedule}")   # CRON式やインターバル
print(f"定義: {task_detail.definition}")          # 実行するSQL
```

---

## 8-5 まとめ：Python から一連のパイプライン管理スクリプト

> 💡 **このスクリプトで何ができるか？**
> CSV のアップロード → パイプライン実行 → タスク状態確認 を
> 1つのPythonスクリプトで完結させます。
> CI/CDやバッチジョブからこのスクリプトを呼び出すことで、
> 全て自動化されたデータパイプラインが構築できます。

```python
"""
pipeline_manager.py
Snowflake Python APIs を使った Bronze/Silver/Gold パイプライン管理スクリプト
"""
import snowflake.connector
from snowflake.core import Root
import glob

# 接続情報を一箇所にまとめる（本番では環境変数や秘密管理サービスから取得する）
CONN_PARAMS = {
    "account": "<アカウント識別子>",   # Snowflakeのアカウント識別子
    "user": "<ユーザー名>",
    "password": "<パスワード>",         # ⚠️ 本番ではコードに直書きしないこと
    "warehouse": "COMPUTE_WH",
    "database": "SENSOR_LAB",
    "schema": "BRONZE"
}


def upload_csv_files(conn, file_pattern: str, stage: str):
    """CSVをステージへアップロードする関数"""
    cur = conn.cursor()
    files = glob.glob(file_pattern)     # ワイルドカードにマッチするファイルを収集
    for f in files:
        f_escaped = f.replace("\\", "/")  # WindowsパスをSnowflake互換スラッシュに変換
        # OVERWRITE=TRUE: 既に同名ファイルがあっても上書き
        cur.execute(f"PUT file://{f_escaped} @{stage} AUTO_COMPRESS=FALSE OVERWRITE=TRUE")
        print(f"[PUT] {f}")
    cur.close()


def run_full_pipeline(conn):
    """フルパイプライン実行（Bronze→Silver→Gold を1コールで実行）"""
    cur = conn.cursor()
    cur.execute("CALL SENSOR_LAB.BRONZE.SP_FULL_PIPELINE()")
    result = cur.fetchone()[0]  # プロシージャの戻り値（各ステップの結果メッセージ）を取得
    print(f"[PIPELINE] {result}")
    cur.close()


def check_task_status(root: Root):
    """全スキーマのタスク状態を一覧表示する"""
    for schema in ["BRONZE", "SILVER", "GOLD"]:
        tasks = root.databases["SENSOR_LAB"].schemas[schema].tasks.iter()
        for t in tasks:
            # state: started（有効） / suspended（停止）
            print(f"[TASK] {schema}.{t.name}: {t.state}")


if __name__ == "__main__":
    # 接続確立
    conn = snowflake.connector.connect(**CONN_PARAMS)
    root = Root(conn)  # Python APIs のルートオブジェクト

    # Step 1: CSVをアップロード（ステージへ転送）
    upload_csv_files(conn, "C:\\path\\to\\sensor_*.csv", "SENSOR_LAB.BRONZE.BRONZE_STAGE")

    # Step 2: パイプライン実行（Bronze→Silver→Gold の変換・集計）
    run_full_pipeline(conn)

    # Step 3: タスク状態確認（定期実行が正常に動作しているか確認）
    check_task_status(root)

    conn.close()  # 接続を閉じる
```

---

## チェックリスト

- [ ] Python APIs と Connector の使い分けを説明できる
- [ ] `root.databases[...].schemas[...].tables.iter()` でテーブル一覧を取得できた
- [ ] `fetch()` でテーブルのカラム詳細を取得できた
- [ ] Python APIs でTaskをResume/Suspendできた
- [ ] `pipeline_manager.py` でアップロード〜パイプライン実行〜状態確認を一括実行できた

---

## 総まとめチェックリスト

全Chapterを終えたら、以下が全て説明・実行できるか確認しましょう。

| Chapter | 確認項目 |
|---|---|
| Ch.0 | Snowsight・SnowSQL・Pythonの接続、3層スキーマの作成 |
| Ch.1 | Warehouse課金の仕組み、メダリオンアーキテクチャの意義 |
| Ch.2 | ステージの種類・役割、PUTコマンドでのCSVアップロード |
| Ch.3 | COPY INTOの基本、べき等性（重複ロード防止）の仕組み |
| Ch.4 | クレンジングロジック、QUALIFYによる重複排除 |
| Ch.5 | Gold層の意義、COUNT(DISTINCT)の使いどころ |
| Ch.6 | ストアドプロシージャの作成・TRUNCATE+INSERTパターン |
| Ch.7 | Task Treeの依存関係、RESUME順序の罠 |
| Ch.8 | Python APIsでのオブジェクト管理・パイプライン自動化 |
