# Snowflake センサーデータ取込・自動化 習得カリキュラム

> **対象者**: Pythonは業務経験あり・Snowflakeはほぼ未経験  
> **期間**: 1〜2週間（1日30分〜1時間）  
> **スタイル**: ハンズオン中心・テーマ別チャプター形式  
> **OS**: Windows  
> **使用ツール**: Snowsight / SnowSQL / Python（Connector + APIs）

---

## 全体マップ

```
Chapter 0  環境セットアップ
    ↓
Chapter 1  Snowflake基本概念 & Snowsight操作
    ↓
Chapter 2  内部ステージ & PUT（ファイルアップロード）
    ↓
Chapter 3  COPY INTO でBronze層ロード
    ↓
Chapter 4  Silver層：クレンジング & 変換
    ↓
Chapter 5  Gold層：集計 & 分析
    ↓
Chapter 6  ストアドプロシージャ（Bronze→Silver→Gold 一括処理）
    ↓
Chapter 7  Task による定期自動実行
    ↓
Chapter 8  Snowflake Python APIs でオブジェクト管理
```

---

## 練習用CSVファイル一覧

| ファイル名 | 拠点 | 日付 | 行数 | 用途 |
|---|---|---|---|---|
| sensor_tokyo_20240115.csv | 東京 | 2024-01-15 | 432 | 基本ロード練習 |
| sensor_tokyo_20240116.csv | 東京 | 2024-01-16 | 432 | 追加ロード練習 |
| sensor_osaka_20240115.csv | 大阪 | 2024-01-15 | 432 | 複数拠点練習 |
| sensor_osaka_20240116.csv | 大阪 | 2024-01-16 | 432 | 複数拠点練習 |
| sensor_nagoya_20240115.csv | 名古屋 | 2024-01-15 | 288 | 複数拠点練習 |
| sensor_nagoya_20240116.csv | 名古屋 | 2024-01-16 | 288 | 複数拠点練習 |
| sensor_fukuoka_20240115.csv | 福岡 | 2024-01-15 | 288 | 複数拠点練習 |
| sensor_fukuoka_20240116.csv | 福岡 | 2024-01-16 | 288 | 複数拠点練習 |
| sensor_all_20240117.csv | 全拠点 | 2024-01-17 | 1440 | 複数ファイル一括ロード |
| sensor_dirty_20240118.csv | 全拠点 | 2024-01-18 | 1440 | **Silver層クレンジング用（NULL・外れ値あり）** |

**CSVスキーマ**:
```
timestamp        TIMESTAMP  例: 2024-01-15 00:00:00
sensor_id        VARCHAR    例: S001
location         VARCHAR    例: tokyo
temperature_c    FLOAT      気温（℃）※dirty版はNULL・異常値あり
humidity_pct     FLOAT      湿度（%）※dirty版はNULL・異常値あり
battery_pct      FLOAT      バッテリー残量（%）
status           VARCHAR    OK / ERROR
```

---

## Chapter 0：環境セットアップ

**目安時間**: 30〜60分  
**ゴール**: 学習環境を整え、最初のSQLを実行できる

### 0-1 Snowsightにログイン
1. https://app.snowflake.com にアクセス
2. アカウントURLを入力してログイン
3. 左メニュー「Worksheets」を開き、新規ワークシート作成
4. 動作確認クエリを実行：
```sql
SELECT CURRENT_USER(), CURRENT_ROLE(), CURRENT_WAREHOUSE();
```

### 0-2 SnowSQL（CLI）のインストール（Windows）
1. https://docs.snowflake.com/en/user-guide/snowsql-install-config からインストーラをDL
2. インストール後、コマンドプロンプトで確認：
```cmd
snowsql --version
```
3. 接続テスト：
```cmd
snowsql -a <アカウント識別子> -u <ユーザー名>
```

### 0-3 Python環境の準備
```cmd
pip install snowflake-connector-python
pip install snowflake-snowpark-python
pip install "snowflake[core]"   # Snowflake Python APIs
```

動作確認（Python）：
```python
import snowflake.connector

conn = snowflake.connector.connect(
    account="<アカウント識別子>",
    user="<ユーザー名>",
    password="<パスワード>",
    warehouse="COMPUTE_WH",
    database="<DB名>",
    schema="PUBLIC"
)
cur = conn.cursor()
cur.execute("SELECT CURRENT_VERSION()")
print(cur.fetchone())
conn.close()
```

### 0-4 学習用DB・スキーマの作成（Snowsight）
```sql
-- 学習専用データベースを作成
CREATE DATABASE IF NOT EXISTS SENSOR_LAB;

-- 3層アーキテクチャ用スキーマ
CREATE SCHEMA IF NOT EXISTS SENSOR_LAB.BRONZE;
CREATE SCHEMA IF NOT EXISTS SENSOR_LAB.SILVER;
CREATE SCHEMA IF NOT EXISTS SENSOR_LAB.GOLD;

USE DATABASE SENSOR_LAB;
USE SCHEMA BRONZE;
```

---

## Chapter 1：Snowflake基本概念 & Snowsight操作

**目安時間**: 60〜90分  
**ゴール**: Snowflakeの主要概念を理解し、Snowsightを使いこなせる

### 1-1 主要概念の整理（読むだけ5分）

| 概念 | 説明 |
|---|---|
| **Organization** | 会社単位の最上位 |
| **Account** | Snowflakeの1インスタンス |
| **Database** | テーブルの入れ物 |
| **Schema** | Database内の名前空間 |
| **Warehouse** | クエリを実行するコンピュートリソース（課金対象） |
| **Role** | 権限管理の単位 |
| **Stage** | ファイルの一時置き場 |

### 1-2 Snowsight 画面の基本操作

**ワークシート操作**:
- 左メニュー「＋」→「SQL Worksheet」で新規作成
- Ctrl+Enter でカーソル行のSQLを実行
- Ctrl+Shift+Enter で全文実行

**確認クエリ集**:
```sql
-- オブジェクト一覧確認
SHOW DATABASES;
SHOW SCHEMAS IN DATABASE SENSOR_LAB;
SHOW WAREHOUSES;
SHOW ROLES;

-- 現在のコンテキスト確認
SELECT CURRENT_DATABASE(), CURRENT_SCHEMA(), CURRENT_WAREHOUSE();
```

### 1-3 テーブルの手動作成と操作
```sql
USE SCHEMA SENSOR_LAB.BRONZE;

-- Bronze層のテーブルを作成
CREATE OR REPLACE TABLE RAW_SENSOR (
    timestamp       TIMESTAMP_NTZ,
    sensor_id       VARCHAR(10),
    location        VARCHAR(50),
    temperature_c   FLOAT,
    humidity_pct    FLOAT,
    battery_pct     FLOAT,
    status          VARCHAR(10),
    _loaded_at      TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    _source_file    VARCHAR(255)
);

-- テーブル構造確認
DESCRIBE TABLE RAW_SENSOR;
```

### 1-4 Snowsight でのデータプレビュー
- 左メニュー「Data」→「Databases」→「SENSOR_LAB」→「BRONZE」→「RAW_SENSOR」
- 「Preview」タブでデータ確認
- 「Columns」タブでスキーマ確認

---

## Chapter 2：内部ステージ & PUT（ファイルアップロード）

**目安時間**: 60〜90分  
**ゴール**: ローカルCSVを内部ステージへアップロードできる

### 2-1 内部ステージの種類（読むだけ3分）

| 種類 | 用途 |
|---|---|
| ユーザーステージ `@~` | 自分専用 |
| テーブルステージ `@%テーブル名` | テーブル専用 |
| **名前付きステージ** `@ステージ名` | チームで共有・推奨 |

### 2-2 名前付きステージの作成（Snowsight）
```sql
USE SCHEMA SENSOR_LAB.BRONZE;

-- File Format（CSV定義）を先に作成
CREATE OR REPLACE FILE FORMAT CSV_FORMAT
    TYPE = 'CSV'
    FIELD_DELIMITER = ','
    RECORD_DELIMITER = '\n'
    SKIP_HEADER = 1
    NULL_IF = ('', 'NULL', 'null')
    EMPTY_FIELD_AS_NULL = TRUE
    COMPRESSION = 'AUTO';

-- 名前付きステージを作成
CREATE OR REPLACE STAGE BRONZE_STAGE
    FILE_FORMAT = CSV_FORMAT
    COMMENT = 'センサーCSVファイルのアップロード先';

-- ステージ確認
SHOW STAGES;
```

### 2-3 SnowSQL から PUT でアップロード
```cmd
snowsql -a <アカウント識別子> -u <ユーザー名>
```

SnowSQL接続後：
```sql
-- データベース・スキーマを選択
USE DATABASE SENSOR_LAB;
USE SCHEMA BRONZE;

-- 1ファイルをアップロード
PUT file://C:\path\to\sensor_tokyo_20240115.csv @BRONZE_STAGE AUTO_COMPRESS=FALSE;

-- 複数ファイルを一括アップロード（ワイルドカード）
PUT file://C:\path\to\sensor_*.csv @BRONZE_STAGE AUTO_COMPRESS=FALSE;

-- ステージ上のファイル確認
LIST @BRONZE_STAGE;
```

### 2-4 Python（Snowflake Connector）から PUT
```python
import snowflake.connector
import glob

conn = snowflake.connector.connect(
    account="<アカウント識別子>",
    user="<ユーザー名>",
    password="<パスワード>",
    warehouse="COMPUTE_WH",
    database="SENSOR_LAB",
    schema="BRONZE"
)
cur = conn.cursor()

# 1ファイルアップロード
cur.execute("PUT file://C:\\path\\to\\sensor_tokyo_20240115.csv @BRONZE_STAGE AUTO_COMPRESS=FALSE")

# 複数ファイルをループでアップロード
files = glob.glob("C:\\path\\to\\sensor_*.csv")
for f in files:
    f_escaped = f.replace("\\", "/")
    cur.execute(f"PUT file://{f_escaped} @BRONZE_STAGE AUTO_COMPRESS=FALSE OVERWRITE=TRUE")
    print(f"アップロード完了: {f}")

# ステージ上のファイル確認
cur.execute("LIST @BRONZE_STAGE")
for row in cur.fetchall():
    print(row)

conn.close()
```

### 2-5 確認ポイント（Snowsight）
```sql
-- ステージのファイル一覧
LIST @BRONZE_STAGE;

-- ステージ上のCSVをプレビュー（ロード前確認）
SELECT $1, $2, $3, $4, $5
FROM @BRONZE_STAGE/sensor_tokyo_20240115.csv
LIMIT 5;
```

---

## Chapter 3：COPY INTO でBronze層ロード

**目安時間**: 60〜90分  
**ゴール**: ステージのCSVをテーブルへロードし、Bronze層を完成させる

### 3-1 基本的な COPY INTO
```sql
USE SCHEMA SENSOR_LAB.BRONZE;

-- 1ファイルをロード
COPY INTO RAW_SENSOR (timestamp, sensor_id, location, temperature_c, humidity_pct, battery_pct, status, _source_file)
FROM (
    SELECT $1, $2, $3, $4, $5, $6, $7,
           METADATA$FILENAME  -- ソースファイル名を自動記録
    FROM @BRONZE_STAGE/sensor_tokyo_20240115.csv
)
FILE_FORMAT = (FORMAT_NAME = 'CSV_FORMAT')
ON_ERROR = 'CONTINUE';  -- エラー行をスキップして続行

-- ロード結果確認
SELECT * FROM RAW_SENSOR LIMIT 10;
SELECT COUNT(*) FROM RAW_SENSOR;
```

### 3-2 全ファイルを一括ロード（パターンマッチ）
```sql
-- ステージ上の全CSVを一括ロード
COPY INTO RAW_SENSOR (timestamp, sensor_id, location, temperature_c, humidity_pct, battery_pct, status, _source_file)
FROM (
    SELECT $1, $2, $3, $4, $5, $6, $7,
           METADATA$FILENAME
    FROM @BRONZE_STAGE
)
FILE_FORMAT = (FORMAT_NAME = 'CSV_FORMAT')
PATTERN = '.*sensor_.*\.csv'
ON_ERROR = 'CONTINUE';
```

### 3-3 ロードエラーの確認
```sql
-- 直近のロード履歴を確認
SELECT *
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME => 'RAW_SENSOR',
    START_TIME => DATEADD(HOURS, -1, CURRENT_TIMESTAMP())
));

-- エラー内容の詳細確認
SELECT *
FROM TABLE(VALIDATE(RAW_SENSOR, JOB_ID => '<JOB_ID>'));
```

### 3-4 ロード状況の確認（Snowsight）
- 左メニュー「Monitoring」→「Query History」でCOPY INTOの実行履歴を確認
- 実行時間・行数・エラーをGUIで確認できる

### 3-5 重複ロード防止（べき等性）
```sql
-- Snowflakeは同一ファイルの重複ロードをデフォルトでスキップする
-- （ファイルのメタデータを内部で管理）
-- 強制的に再ロードしたい場合：
COPY INTO RAW_SENSOR ...
FORCE = TRUE;  -- 重複チェックをスキップ
```

---

## Chapter 4：Silver層 — クレンジング & 変換

**目安時間**: 90〜120分  
**ゴール**: Bronze の生データをクレンジングし、Silver層を構築する

### 4-1 Silver層テーブルの作成
```sql
USE SCHEMA SENSOR_LAB.SILVER;

CREATE OR REPLACE TABLE SENSOR_CLEAN (
    sensor_id       VARCHAR(10),
    location        VARCHAR(50),
    recorded_at     TIMESTAMP_NTZ,
    recorded_date   DATE,
    recorded_hour   INT,
    temperature_c   FLOAT,
    humidity_pct    FLOAT,
    battery_pct     FLOAT,
    status          VARCHAR(10),
    is_anomaly      BOOLEAN,  -- 外れ値フラグ
    _loaded_at      TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    _source_file    VARCHAR(255)
);
```

### 4-2 Bronze → Silver 変換SQL（クレンジングロジック）
```sql
INSERT INTO SENSOR_LAB.SILVER.SENSOR_CLEAN
SELECT
    sensor_id,
    location,
    timestamp                               AS recorded_at,
    DATE(timestamp)                         AS recorded_date,
    HOUR(timestamp)                         AS recorded_hour,

    -- NULLの場合は除外・外れ値は境界値でクランプ
    CASE
        WHEN temperature_c IS NULL     THEN NULL
        WHEN temperature_c < -50       THEN NULL  -- センサー異常値
        WHEN temperature_c > 80        THEN NULL
        ELSE temperature_c
    END                                     AS temperature_c,

    CASE
        WHEN humidity_pct IS NULL      THEN NULL
        WHEN humidity_pct < 0          THEN 0
        WHEN humidity_pct > 100        THEN 100
        ELSE humidity_pct
    END                                     AS humidity_pct,

    battery_pct,
    status,

    -- 外れ値フラグ（温度が正常範囲外）
    (temperature_c < -50 OR temperature_c > 80)  AS is_anomaly,

    CURRENT_TIMESTAMP()                     AS _loaded_at,
    _source_file
FROM SENSOR_LAB.BRONZE.RAW_SENSOR
-- 重複排除（同じセンサーID+タイムスタンプは1件のみ）
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY sensor_id, timestamp
    ORDER BY _loaded_at DESC
) = 1;
```

### 4-3 クレンジング結果の確認
```sql
-- NULL件数の確認
SELECT
    COUNT(*)                                    AS total,
    SUM(CASE WHEN temperature_c IS NULL THEN 1 ELSE 0 END) AS temp_nulls,
    SUM(CASE WHEN humidity_pct IS NULL THEN 1 ELSE 0 END)  AS hum_nulls,
    SUM(CASE WHEN is_anomaly THEN 1 ELSE 0 END)            AS anomalies
FROM SENSOR_LAB.SILVER.SENSOR_CLEAN;

-- 拠点・日付ごとのデータ件数確認
SELECT location, recorded_date, COUNT(*) AS cnt
FROM SENSOR_LAB.SILVER.SENSOR_CLEAN
GROUP BY location, recorded_date
ORDER BY location, recorded_date;
```

### 4-4 VIEWの作成（正常データのみ）
```sql
-- 正常データのみのビュー（分析に使う）
CREATE OR REPLACE VIEW SENSOR_LAB.SILVER.V_SENSOR_VALID AS
SELECT *
FROM SENSOR_LAB.SILVER.SENSOR_CLEAN
WHERE temperature_c IS NOT NULL
  AND humidity_pct IS NOT NULL
  AND status = 'OK'
  AND NOT is_anomaly;
```

---

## Chapter 5：Gold層 — 集計 & 分析

**目安時間**: 60〜90分  
**ゴール**: Silver層から集計テーブルを作りビジネス価値のあるデータを整備する

### 5-1 Gold層テーブルの設計
```sql
USE SCHEMA SENSOR_LAB.GOLD;

-- 1時間ごとの集計テーブル
CREATE OR REPLACE TABLE HOURLY_STATS (
    location        VARCHAR(50),
    stat_date       DATE,
    stat_hour       INT,
    sensor_count    INT,
    avg_temp        FLOAT,
    min_temp        FLOAT,
    max_temp        FLOAT,
    avg_humidity    FLOAT,
    min_humidity    FLOAT,
    max_humidity    FLOAT,
    anomaly_count   INT,
    _loaded_at      TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- 日次の集計テーブル
CREATE OR REPLACE TABLE DAILY_STATS (
    location        VARCHAR(50),
    stat_date       DATE,
    sensor_count    INT,
    avg_temp        FLOAT,
    min_temp        FLOAT,
    max_temp        FLOAT,
    avg_humidity    FLOAT,
    min_humidity    FLOAT,
    max_humidity    FLOAT,
    anomaly_count   INT,
    _loaded_at      TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);
```

### 5-2 Silver → Gold 集計SQL
```sql
-- 1時間ごとの集計
INSERT INTO SENSOR_LAB.GOLD.HOURLY_STATS
SELECT
    location,
    recorded_date               AS stat_date,
    recorded_hour               AS stat_hour,
    COUNT(DISTINCT sensor_id)   AS sensor_count,
    ROUND(AVG(temperature_c),2) AS avg_temp,
    MIN(temperature_c)          AS min_temp,
    MAX(temperature_c)          AS max_temp,
    ROUND(AVG(humidity_pct),2)  AS avg_humidity,
    MIN(humidity_pct)           AS min_humidity,
    MAX(humidity_pct)           AS max_humidity,
    SUM(CASE WHEN is_anomaly THEN 1 ELSE 0 END) AS anomaly_count,
    CURRENT_TIMESTAMP()         AS _loaded_at
FROM SENSOR_LAB.SILVER.V_SENSOR_VALID
GROUP BY location, recorded_date, recorded_hour;

-- 日次集計
INSERT INTO SENSOR_LAB.GOLD.DAILY_STATS
SELECT
    location,
    stat_date,
    SUM(sensor_count)           AS sensor_count,
    ROUND(AVG(avg_temp),2)      AS avg_temp,
    MIN(min_temp)               AS min_temp,
    MAX(max_temp)               AS max_temp,
    ROUND(AVG(avg_humidity),2)  AS avg_humidity,
    MIN(min_humidity)           AS min_humidity,
    MAX(max_humidity)           AS max_humidity,
    SUM(anomaly_count)          AS anomaly_count,
    CURRENT_TIMESTAMP()         AS _loaded_at
FROM SENSOR_LAB.GOLD.HOURLY_STATS
GROUP BY location, stat_date;
```

### 5-3 分析クエリ例
```sql
-- 拠点別の日次平均気温ランキング
SELECT location, stat_date, avg_temp
FROM SENSOR_LAB.GOLD.DAILY_STATS
ORDER BY stat_date, avg_temp DESC;

-- 異常検知が多い拠点TOP
SELECT location, SUM(anomaly_count) AS total_anomalies
FROM SENSOR_LAB.GOLD.DAILY_STATS
GROUP BY location
ORDER BY total_anomalies DESC;
```

### 5-4 Snowsight でのチャート作成
1. 上記SQLをワークシートで実行
2. 結果パネル右上「Chart」タブをクリック
3. X軸: `STAT_DATE`、Y軸: `AVG_TEMP`、カラー: `LOCATION` を設定
4. 折れ線グラフで拠点別気温推移を可視化

---

## Chapter 6：ストアドプロシージャ

**目安時間**: 90〜120分  
**ゴール**: Bronze→Silver→Gold の処理をストアドプロシージャにまとめる

### 6-1 ストアドプロシージャの基本構文（JavaScript）
```sql
CREATE OR REPLACE PROCEDURE SENSOR_LAB.BRONZE.SP_LOAD_FROM_STAGE(
    stage_name  VARCHAR,
    file_pattern VARCHAR
)
RETURNS VARCHAR
LANGUAGE JAVASCRIPT
EXECUTE AS CALLER
AS $$
    var result_msg = "";
    try {
        // COPY INTO でステージからBronzeへロード
        var sql = `
            COPY INTO SENSOR_LAB.BRONZE.RAW_SENSOR
            (timestamp, sensor_id, location, temperature_c, humidity_pct, battery_pct, status, _source_file)
            FROM (
                SELECT $1,$2,$3,$4,$5,$6,$7, METADATA$FILENAME
                FROM @SENSOR_LAB.BRONZE.` + STAGE_NAME + `
            )
            FILE_FORMAT = (FORMAT_NAME = 'SENSOR_LAB.BRONZE.CSV_FORMAT')
            PATTERN = '` + FILE_PATTERN + `'
            ON_ERROR = 'CONTINUE'
        `;
        var stmt = snowflake.execute({sqlText: sql});
        result_msg = "Bronze load completed.";
    } catch(err) {
        result_msg = "ERROR: " + err.message;
    }
    return result_msg;
$$;

-- 実行テスト
CALL SENSOR_LAB.BRONZE.SP_LOAD_FROM_STAGE('BRONZE_STAGE', '.*sensor_.*\\.csv');
```

### 6-2 Silver層変換のストアドプロシージャ
```sql
CREATE OR REPLACE PROCEDURE SENSOR_LAB.SILVER.SP_TRANSFORM_TO_SILVER()
RETURNS VARCHAR
LANGUAGE JAVASCRIPT
EXECUTE AS CALLER
AS $$
    try {
        // Silver層を一旦クリアして再作成（TRUNCATE + INSERT）
        snowflake.execute({sqlText: "TRUNCATE TABLE SENSOR_LAB.SILVER.SENSOR_CLEAN"});

        var sql = `
            INSERT INTO SENSOR_LAB.SILVER.SENSOR_CLEAN
            SELECT
                sensor_id, location, timestamp,
                DATE(timestamp), HOUR(timestamp),
                CASE WHEN temperature_c < -50 OR temperature_c > 80 THEN NULL
                     ELSE temperature_c END,
                CASE WHEN humidity_pct < 0 THEN 0
                     WHEN humidity_pct > 100 THEN 100
                     ELSE humidity_pct END,
                battery_pct, status,
                (temperature_c < -50 OR temperature_c > 80),
                CURRENT_TIMESTAMP(), _source_file
            FROM SENSOR_LAB.BRONZE.RAW_SENSOR
            QUALIFY ROW_NUMBER() OVER (
                PARTITION BY sensor_id, timestamp ORDER BY _loaded_at DESC
            ) = 1
        `;
        snowflake.execute({sqlText: sql});
        return "Silver transform completed.";
    } catch(err) {
        return "ERROR: " + err.message;
    }
$$;
```

### 6-3 Gold層集計のストアドプロシージャ
```sql
CREATE OR REPLACE PROCEDURE SENSOR_LAB.GOLD.SP_AGGREGATE_TO_GOLD()
RETURNS VARCHAR
LANGUAGE JAVASCRIPT
EXECUTE AS CALLER
AS $$
    try {
        snowflake.execute({sqlText: "TRUNCATE TABLE SENSOR_LAB.GOLD.HOURLY_STATS"});
        snowflake.execute({sqlText: "TRUNCATE TABLE SENSOR_LAB.GOLD.DAILY_STATS"});

        snowflake.execute({sqlText: `
            INSERT INTO SENSOR_LAB.GOLD.HOURLY_STATS
            SELECT location, recorded_date, recorded_hour,
                COUNT(DISTINCT sensor_id), ROUND(AVG(temperature_c),2),
                MIN(temperature_c), MAX(temperature_c),
                ROUND(AVG(humidity_pct),2), MIN(humidity_pct), MAX(humidity_pct),
                SUM(CASE WHEN is_anomaly THEN 1 ELSE 0 END),
                CURRENT_TIMESTAMP()
            FROM SENSOR_LAB.SILVER.V_SENSOR_VALID
            GROUP BY location, recorded_date, recorded_hour
        `});

        snowflake.execute({sqlText: `
            INSERT INTO SENSOR_LAB.GOLD.DAILY_STATS
            SELECT location, stat_date,
                SUM(sensor_count), ROUND(AVG(avg_temp),2),
                MIN(min_temp), MAX(max_temp),
                ROUND(AVG(avg_humidity),2), MIN(min_humidity), MAX(max_humidity),
                SUM(anomaly_count), CURRENT_TIMESTAMP()
            FROM SENSOR_LAB.GOLD.HOURLY_STATS
            GROUP BY location, stat_date
        `});

        return "Gold aggregation completed.";
    } catch(err) {
        return "ERROR: " + err.message;
    }
$$;
```

### 6-4 3層まとめて実行するマスタープロシージャ
```sql
CREATE OR REPLACE PROCEDURE SENSOR_LAB.BRONZE.SP_FULL_PIPELINE()
RETURNS VARCHAR
LANGUAGE JAVASCRIPT
EXECUTE AS CALLER
AS $$
    var results = [];

    var procs = [
        "CALL SENSOR_LAB.BRONZE.SP_LOAD_FROM_STAGE('BRONZE_STAGE', '.*sensor_.*\\.csv')",
        "CALL SENSOR_LAB.SILVER.SP_TRANSFORM_TO_SILVER()",
        "CALL SENSOR_LAB.GOLD.SP_AGGREGATE_TO_GOLD()"
    ];

    for (var i = 0; i < procs.length; i++) {
        try {
            var res = snowflake.execute({sqlText: procs[i]});
            res.next();
            results.push(res.getColumnValue(1));
        } catch(err) {
            results.push("STEP " + i + " ERROR: " + err.message);
            break;
        }
    }
    return results.join(" | ");
$$;

-- 全パイプライン実行テスト
CALL SENSOR_LAB.BRONZE.SP_FULL_PIPELINE();
```

---

## Chapter 7：Task による定期自動実行

**目安時間**: 60〜90分  
**ゴール**: ストアドプロシージャをTaskに登録し定期実行できるようにする

### 7-1 Taskの基本
```sql
-- Task作成（毎時0分に実行）
CREATE OR REPLACE TASK SENSOR_LAB.BRONZE.TASK_FULL_PIPELINE
    WAREHOUSE = COMPUTE_WH
    SCHEDULE = 'USING CRON 0 * * * * Asia/Tokyo'  -- 毎時0分
    -- SCHEDULE = '5 MINUTE'  -- 分単位指定も可能
AS
    CALL SENSOR_LAB.BRONZE.SP_FULL_PIPELINE();
```

### 7-2 Task Tree（依存関係を持つTask）
```sql
-- 親Task：Bronzeロード
CREATE OR REPLACE TASK SENSOR_LAB.BRONZE.TASK_BRONZE
    WAREHOUSE = COMPUTE_WH
    SCHEDULE = 'USING CRON 0 * * * * Asia/Tokyo'
AS
    CALL SENSOR_LAB.BRONZE.SP_LOAD_FROM_STAGE('BRONZE_STAGE', '.*sensor_.*\\.csv');

-- 子Task：Silverへ変換（親完了後に実行）
-- ※ AFTER で依存関係を持つTaskは同じスキーマに配置する必要があるため BRONZE スキーマに統一
CREATE OR REPLACE TASK SENSOR_LAB.BRONZE.TASK_SILVER
    WAREHOUSE = COMPUTE_WH
    AFTER SENSOR_LAB.BRONZE.TASK_BRONZE
AS
    CALL SENSOR_LAB.SILVER.SP_TRANSFORM_TO_SILVER();

-- 孫Task：Goldへ集計（子完了後に実行）
CREATE OR REPLACE TASK SENSOR_LAB.BRONZE.TASK_GOLD
    WAREHOUSE = COMPUTE_WH
    AFTER SENSOR_LAB.BRONZE.TASK_SILVER
AS
    CALL SENSOR_LAB.GOLD.SP_AGGREGATE_TO_GOLD();
```

### 7-3 Task の有効化・無効化
```sql
-- Task は作成後デフォルトで SUSPENDED（停止中）
-- 子Taskから先に有効化すること！（重要）
ALTER TASK SENSOR_LAB.BRONZE.TASK_GOLD RESUME;
ALTER TASK SENSOR_LAB.BRONZE.TASK_SILVER RESUME;
ALTER TASK SENSOR_LAB.BRONZE.TASK_BRONZE RESUME;  -- 最後に親を有効化

-- 確認
SHOW TASKS IN DATABASE SENSOR_LAB;

-- 停止する場合（子Taskが先に停止できない）
ALTER TASK SENSOR_LAB.BRONZE.TASK_BRONZE SUSPEND;

-- 即時手動実行（テスト用）
EXECUTE TASK SENSOR_LAB.BRONZE.TASK_BRONZE;
```

### 7-4 Task 実行履歴の確認（Snowsight）
```sql
-- 実行履歴をSQLで確認
SELECT *
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
    TASK_NAME => 'TASK_BRONZE',
    SCHEDULED_TIME_RANGE_START => DATEADD(HOURS, -24, CURRENT_TIMESTAMP())
))
ORDER BY SCHEDULED_TIME DESC;
```

**Snowsight GUI で確認する方法**:
1. 左メニュー「Monitoring」→「Task History」
2. タスク名・状態・実行時間・エラーをGUIで確認
3. Task Graph（Taskツリーの依存関係図）も確認できる

---

## Chapter 8：Snowflake Python APIs でオブジェクト管理

**目安時間**: 60〜90分  
**ゴール**: PythonコードからSnowflakeのオブジェクト（テーブル・タスクなど）を管理できる

### 8-1 Python APIs の初期化
```python
from snowflake.core import Root
from snowflake.core.database import Database
from snowflake.core.schema import Schema
from snowflake.core.table import Table, TableColumn
from snowflake.core.task import Task, StoredProcedureCall
import snowflake.connector

# コネクション確立
conn = snowflake.connector.connect(
    account="<アカウント識別子>",
    user="<ユーザー名>",
    password="<パスワード>",
    warehouse="COMPUTE_WH",
    database="SENSOR_LAB",
    schema="BRONZE"
)

root = Root(conn)
```

### 8-2 データベース・スキーマの操作
```python
# データベース一覧取得
databases = root.databases.iter()
for db in databases:
    print(db.name)

# スキーマ一覧
schemas = root.databases["SENSOR_LAB"].schemas.iter()
for s in schemas:
    print(s.name)

# 新しいスキーマを作成
new_schema = Schema(name="ARCHIVE")
root.databases["SENSOR_LAB"].schemas.create(new_schema, mode="if_not_exists")
print("スキーマ ARCHIVE を作成しました")
```

### 8-3 テーブルのオブジェクト管理
```python
# テーブル一覧取得
tables = root.databases["SENSOR_LAB"].schemas["BRONZE"].tables.iter()
for t in tables:
    print(f"テーブル: {t.name}")

# テーブルの詳細取得
table = root.databases["SENSOR_LAB"].schemas["BRONZE"].tables["RAW_SENSOR"].fetch()
print(f"テーブル名: {table.name}")
print(f"カラム数: {len(table.columns)}")
for col in table.columns:
    print(f"  {col.name}: {col.datatype}")
```

### 8-4 Task の Python API 管理
```python
from snowflake.core.task import Task

# タスク一覧取得
tasks = root.databases["SENSOR_LAB"].schemas["BRONZE"].tasks.iter()
for t in tasks:
    print(f"タスク: {t.name}, 状態: {t.state}")

# タスクを RESUME（有効化）
task_ref = root.databases["SENSOR_LAB"].schemas["BRONZE"].tasks["TASK_BRONZE"]
task_ref.resume()
print("TASK_BRONZE を有効化しました")

# タスクを SUSPEND（停止）
task_ref.suspend()
print("TASK_BRONZE を停止しました")

# タスクの詳細取得
task_detail = task_ref.fetch()
print(f"スケジュール: {task_detail.schedule}")
print(f"定義: {task_detail.definition}")
```

### 8-5 まとめ：Python から一連のパイプライン管理スクリプト
```python
"""
pipeline_manager.py
Snowflake Python APIs を使った Bronze/Silver/Gold パイプライン管理
"""
import snowflake.connector
from snowflake.core import Root
import glob

CONN_PARAMS = {
    "account": "<アカウント識別子>",
    "user": "<ユーザー名>",
    "password": "<パスワード>",
    "warehouse": "COMPUTE_WH",
    "database": "SENSOR_LAB",
    "schema": "BRONZE"
}

def upload_csv_files(conn, file_pattern: str, stage: str):
    """CSVをステージへアップロード"""
    cur = conn.cursor()
    files = glob.glob(file_pattern)
    for f in files:
        f_escaped = f.replace("\\", "/")
        cur.execute(f"PUT file://{f_escaped} @{stage} AUTO_COMPRESS=FALSE OVERWRITE=TRUE")
        print(f"[PUT] {f}")
    cur.close()

def run_full_pipeline(conn):
    """フルパイプライン実行"""
    cur = conn.cursor()
    cur.execute("CALL SENSOR_LAB.BRONZE.SP_FULL_PIPELINE()")
    result = cur.fetchone()[0]
    print(f"[PIPELINE] {result}")
    cur.close()

def check_task_status(root: Root):
    """タスク状態確認"""
    for schema in ["BRONZE", "SILVER", "GOLD"]:
        tasks = root.databases["SENSOR_LAB"].schemas[schema].tasks.iter()
        for t in tasks:
            print(f"[TASK] {schema}.{t.name}: {t.state}")

if __name__ == "__main__":
    conn = snowflake.connector.connect(**CONN_PARAMS)
    root = Root(conn)

    # 1. CSVをアップロード
    upload_csv_files(conn, "C:\\path\\to\\sensor_*.csv", "SENSOR_LAB.BRONZE.BRONZE_STAGE")

    # 2. パイプライン実行
    run_full_pipeline(conn)

    # 3. タスク状態確認
    check_task_status(root)

    conn.close()
```

---

## 総まとめチェックリスト

### Chapter 0 ✅
- [ ] Snowsightにログインして最初のSQLを実行できた
- [ ] SnowSQLをWindowsにインストールして接続できた
- [ ] Pythonからsnowflake-connector-pythonで接続できた
- [ ] SENSOR_LAB の3層スキーマを作成できた

### Chapter 1 ✅
- [ ] Database / Schema / Warehouse / Role の概念を説明できる
- [ ] SnowsightのワークシートでSQLを実行できた
- [ ] RAW_SENSORテーブルを作成できた

### Chapter 2 ✅
- [ ] 名前付きステージとFile Formatを作成できた
- [ ] SnowSQL の PUT でCSVをアップロードできた
- [ ] Python の PUT でCSVをアップロードできた
- [ ] LIST @BRONZE_STAGE でファイル確認できた

### Chapter 3 ✅
- [ ] COPY INTO で1ファイルをロードできた
- [ ] パターンマッチで全ファイルを一括ロードできた
- [ ] ロードエラーを COPY_HISTORY で確認できた

### Chapter 4 ✅
- [ ] SENSOR_CLEANテーブルを作成できた
- [ ] NULL・外れ値処理のSQLを書いてSilverにロードできた
- [ ] V_SENSOR_VALID ビューを作成できた

### Chapter 5 ✅
- [ ] HOURLY_STATS / DAILY_STATSテーブルを作成できた
- [ ] 集計SQLを実行してGold層にデータを格納できた
- [ ] Snowsightのチャート機能で可視化できた

### Chapter 6 ✅
- [ ] Bronzeロード・Silver変換・Gold集計それぞれのSPを作成できた
- [ ] SP_FULL_PIPELINE でパイプライン全体を1コールで実行できた

### Chapter 7 ✅
- [ ] Task Treeを作成できた（Bronze→Silver→Gold）
- [ ] TaskをResume/Suspendできた
- [ ] TASK_HISTORYで実行結果を確認できた
- [ ] SnowsightのTask Historyで実行状況を確認できた

### Chapter 8 ✅
- [ ] Python APIs でDB・スキーマ・テーブル一覧を取得できた
- [ ] Python APIs でTaskをResume/Suspendできた
- [ ] pipeline_manager.py でアップロード〜パイプライン実行〜状態確認を一括実行できた

---

## 参考リンク

- [Snowflake ドキュメント（公式）](https://docs.snowflake.com/)
- [COPY INTO コマンドリファレンス](https://docs.snowflake.com/en/sql-reference/sql/copy-into-table)
- [Snowflake Connector for Python](https://docs.snowflake.com/en/developer-guide/python-connector/python-connector)
- [Snowflake Python APIs](https://docs.snowflake.com/en/developer-guide/snowflake-python-api/snowflake-python-api-reference)
- [Tasks and Task Graphs](https://docs.snowflake.com/en/user-guide/tasks-intro)
- [SnowSQL インストールガイド](https://docs.snowflake.com/en/user-guide/snowsql-install-config)
