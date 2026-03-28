# Chapter 2：内部ステージ & PUT（ファイルアップロード）

**目安時間**: 60〜90分
**ゴール**: ローカルCSVを内部ステージへアップロードできる

← [Chapter 1](01_基本概念とSnowsight操作.md) | [目次](README.md) | [Chapter 3 →](03_COPY_INTOでBronze層ロード.md)

---

## このチャプターで何をするか

ローカルPCにあるCSVファイルをSnowflakeに送り込む方法を学びます。
Snowflakeにデータを取り込む流れは **2ステップ** です：

```
① PUT: ローカルCSV → ステージ（Snowflake上の一時置き場）
② COPY INTO: ステージ → テーブル（次のChapterで実施）
```

> 💡 **なぜ2ステップに分かれているのか？**
> ローカルファイルをテーブルに直接入れることはできません。
> まずSnowflake内の「一時置き場（ステージ）」に預けることで、
> Snowflakeが内部で効率よくデータを処理できるようになります。
> また、ステージにファイルを置いておくことで「後から何度でもロード」が可能です。
>
> **やらないと？** COPY INTOコマンドに渡すファイルパスがなくエラーになります。
> **やると？** ファイルをSnowflakeに預けた状態になり、テーブルへのロードが可能になります。

---

## 2-1 内部ステージの種類（読むだけ3分）

| 種類 | 表記 | 用途 | 特徴 |
|---|---|---|---|
| ユーザーステージ | `@~` | 自分専用 | 作成不要・他ユーザーと共有不可 |
| テーブルステージ | `@%テーブル名` | テーブル専用 | 作成不要・テーブルに紐づく |
| **名前付きステージ** | `@ステージ名` | チームで共有 | **作成必要・推奨** |

> 💡 **なぜ名前付きステージが推奨なのか？**
> ユーザーステージやテーブルステージは手軽ですが、
> 「誰がどのファイルを置いたか」の管理や「チームでの共有」が難しくなります。
> 名前付きステージは明示的に作成・管理でき、権限設定もできるため
> チーム開発・本番運用に向いています。
>
> **やらないと？** 個人専用ステージに置いたファイルをチームメンバーが参照できません。
> **やると？** チーム全員が同じステージを使えるようになります。

---

## 2-2 名前付きステージの作成（Snowsight）

> 💡 **File Format（ファイルフォーマット）とは？**
> CSV・JSON・Parquetなど「ファイルの読み方のルール」を定義したオブジェクトです。
> 「区切り文字はカンマ」「1行目はヘッダー」「空文字はNULLとして扱う」などを
> 事前に定義しておくことで、COPY INTO時に毎回同じ設定を書かずに済みます。
>
> **やらないと？** COPY INTOのたびにCSVの読み方を手書きする必要があります。
> **やると？** File Formatを一度作れば再利用でき、設定ミスも防げます。

```sql
USE SCHEMA SENSOR_LAB.BRONZE;

-- File Format（CSV定義）を先に作成
-- ステージはFile Formatを参照するため、先にFile Formatが必要
CREATE OR REPLACE FILE FORMAT CSV_FORMAT
    TYPE = 'CSV'                   -- ファイル種別（CSV / JSON / PARQUET など）
    FIELD_DELIMITER = ','          -- フィールドの区切り文字
    RECORD_DELIMITER = '\n'        -- 行の区切り文字（改行）
    SKIP_HEADER = 1                -- 1行目（ヘッダー行）をスキップ
    NULL_IF = ('', 'NULL', 'null') -- これらの文字列はNULLとして扱う
    EMPTY_FIELD_AS_NULL = TRUE     -- 空フィールドもNULLとして扱う
    COMPRESSION = 'AUTO';          -- 圧縮形式を自動検出（gzip等にも対応）

-- 名前付きステージを作成
CREATE OR REPLACE STAGE BRONZE_STAGE
    FILE_FORMAT = CSV_FORMAT       -- デフォルトで使用するFile Formatを指定
    COMMENT = 'センサーCSVファイルのアップロード先';

-- ステージが作成されたことを確認
SHOW STAGES;
```

---

## 2-3 SnowSQL から PUT でアップロード

> 💡 **PUTコマンドとは？**
> ローカルPCのファイルをSnowflakeのステージに転送するコマンドです。
> Snowsightからは実行できないため、SnowSQLまたはPythonから実行します。
>
> **やらないと？** ローカルCSVがステージに上がらず、COPY INTOができません。
> **やると？** CSVがSnowflakeの内部ストレージに転送され、ロード待ちの状態になります。

まずSnowSQLでSnowflakeに接続します：
```cmd
rem -a: アカウント識別子  -u: ユーザー名（パスワードは対話式で入力）
snowsql -a <アカウント識別子> -u <ユーザー名>
```

SnowSQL接続後、以下のSQLを実行します：
```sql
-- データベース・スキーマを選択（コンテキストを設定しないとオブジェクトが見つからない）
USE DATABASE SENSOR_LAB;
USE SCHEMA BRONZE;

-- 1ファイルをアップロード
-- AUTO_COMPRESS=FALSE: 圧縮せずそのまま転送（小ファイルでは圧縮オーバーヘッドを避ける）
PUT file://C:\path\to\sensor_tokyo_20240115.csv @BRONZE_STAGE AUTO_COMPRESS=FALSE;

-- ワイルドカードで複数ファイルを一括アップロード（sensor_ で始まる全CSV）
PUT file://C:\path\to\sensor_*.csv @BRONZE_STAGE AUTO_COMPRESS=FALSE;

-- ステージ上のファイル一覧を確認（サイズ・更新日時も表示される）
LIST @BRONZE_STAGE;
```

> 💡 **`AUTO_COMPRESS=FALSE` の意味**
> デフォルトでは転送時にgzip圧縮されますが、ファイルが小さい場合は
> 圧縮の処理コストのほうが高くなることがあります。
> 学習用の小さいCSVではFALSEにしておくのが無難です。

---

## 2-4 Python（Snowflake Connector）から PUT

> 💡 **PythonでPUTを使う場面**
> 毎日自動でCSVをアップロードするバッチ処理など、
> 定期実行・自動化が必要な場面ではPythonから実行します。
> SnowSQLは手動実行向き、Pythonはスクリプト化・自動化向きと使い分けます。

```python
import snowflake.connector
import glob  # ファイルのワイルドカード検索に使用

# Snowflakeへの接続
conn = snowflake.connector.connect(
    account="<アカウント識別子>",   # 例: abc12345.ap-northeast-1.aws
    user="<ユーザー名>",
    password="<パスワード>",
    warehouse="COMPUTE_WH",          # ウェアハウス（PUTには使われないが接続時に必要）
    database="SENSOR_LAB",
    schema="BRONZE"
)
cur = conn.cursor()

# 1ファイルアップロード
# Windowsのパスはバックスラッシュ（\）だが、PUTコマンド内はスラッシュ（/）で渡す
cur.execute("PUT file://C:\\path\\to\\sensor_tokyo_20240115.csv @BRONZE_STAGE AUTO_COMPRESS=FALSE")

# 複数ファイルをループでアップロード
files = glob.glob("C:\\path\\to\\sensor_*.csv")  # ワイルドカードで対象ファイルを収集
for f in files:
    f_escaped = f.replace("\\", "/")  # Windowsパスのバックスラッシュをスラッシュに変換
    # OVERWRITE=TRUE: 同名ファイルが既にステージにあっても上書き
    cur.execute(f"PUT file://{f_escaped} @BRONZE_STAGE AUTO_COMPRESS=FALSE OVERWRITE=TRUE")
    print(f"アップロード完了: {f}")

# ステージ上のファイル確認（ファイル名・サイズ・MD5ハッシュが返ってくる）
cur.execute("LIST @BRONZE_STAGE")
for row in cur.fetchall():
    print(row)

conn.close()  # 接続を閉じる（リソースの後始末）
```

---

## 2-5 確認ポイント（Snowsight）

> 💡 **ロード前にステージのデータを目視確認する重要性**
> PUTが成功してもCSVの中身が正しいとは限りません。
> COPY INTOを実行する前に、ステージ上のCSVをプレビューして
> 「カラム数が合っているか」「文字化けがないか」を確認しましょう。

```sql
-- ステージのファイル一覧（ファイルが正しくアップロードされているか確認）
LIST @BRONZE_STAGE;

-- ステージ上のCSVを直接SELECT（ロード前の中身確認）
-- $1, $2, $3 ... は CSVの1列目、2列目、3列目 を意味する（カラム名ではなく位置指定）
SELECT $1, $2, $3, $4, $5
FROM @BRONZE_STAGE/sensor_tokyo_20240115.csv
LIMIT 5;
```

---

## チェックリスト

- [ ] ステージ3種類（ユーザー・テーブル・名前付き）の違いを説明できる
- [ ] 名前付きステージとFile Formatを作成できた
- [ ] SnowSQL の PUT でCSVをアップロードできた
- [ ] Python の PUT でCSVをアップロードできた
- [ ] `LIST @BRONZE_STAGE` でファイル確認できた
- [ ] ステージ上のCSVをSELECTでプレビューできた
