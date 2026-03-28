# Chapter 1：Snowflake基本概念 & Snowsight操作

**目安時間**: 60〜90分
**ゴール**: Snowflakeの主要概念を理解し、Snowsightを使いこなせる

← [Chapter 0](00_環境セットアップ.md) | [目次](README.md) | [Chapter 2 →](02_ステージとPUT.md)

---

## 1-1 主要概念の整理（読むだけ5分）

Snowflakeを使う前に、独自の用語を押さえておきましょう。
知らないと「どこで何を作ればいいか」がわからなくなります。

| 概念 | 説明 | 身近なたとえ |
|---|---|---|
| **Organization** | 会社単位の最上位コンテナ | 会社全体 |
| **Account** | Snowflakeの1インスタンス（1契約） | 会社の1事業部 |
| **Database** | テーブルの入れ物（最大の論理単位） | フォルダ |
| **Schema** | Database内の名前空間（テーブルをグループ化） | サブフォルダ |
| **Warehouse** | クエリを実行するコンピュートリソース | 処理エンジン（CPU・メモリ） |
| **Role** | 権限管理の単位（ユーザーにRoleを付与） | 社内の役職 |
| **Stage** | ファイルのアップロード先一時置き場 | 宅配の荷物受け取りボックス |

> 💡 **Warehouseは「課金の要」**
> Snowflakeはストレージ（データ保存）とコンピュート（処理）が分離しています。
> **データを保存するだけでは課金されません。**
> クエリを実行するときにWarehouseが起動し、その時間に対して課金されます。
>
> **やらないと？** Warehouseを起動したままにすると、使っていない時間も課金されます。
> **やると？** 不要なときに停止（AUTO_SUSPEND）することでコストを抑えられます。

---

## 1-2 Snowsight 画面の基本操作

> 💡 **Snowsightを使いこなす意義**
> Snowsightはクエリ実行だけでなく、実行計画の確認・エラー調査・チャート作成まで
> できる統合コンソールです。使い方を覚えると開発効率が大幅に上がります。

**ワークシート操作**:
- 左メニュー「＋」→「SQL Worksheet」で新規作成
- `Ctrl+Enter` でカーソル行のSQLを実行
- `Ctrl+Shift+Enter` で全文実行

**確認クエリ集**:

```sql
-- 存在するデータベースの一覧を表示
SHOW DATABASES;

-- SENSOR_LAB内のスキーマ一覧を表示
SHOW SCHEMAS IN DATABASE SENSOR_LAB;

-- 利用可能なウェアハウス一覧（サイズ・状態も確認できる）
SHOW WAREHOUSES;

-- 自分に割り当てられたロール一覧
SHOW ROLES;

-- 現在のセッションのコンテキスト確認（DBやスキーマが正しいか確認する習慣を）
SELECT CURRENT_DATABASE(), CURRENT_SCHEMA(), CURRENT_WAREHOUSE();
```

---

## 1-3 テーブルの手動作成と操作

> 💡 **なぜこのタイミングでテーブルを作るのか？**
> データをロードする前にテーブル（受け皿）が必要です。
> Snowflakeは**スキーマオン・ライト**（書くときにスキーマを定義）なので、
> テーブル定義なしにデータを入れることはできません。
>
> **やらないと？** COPY INTOを実行してもロード先がなくエラーになります。
> **やると？** CSVのデータを受け取る「箱」が準備できます。

> 💡 **`_loaded_at` と `_source_file` カラムについて**
> 先頭に `_` がついているカラムは「管理用メタデータ」という慣習です。
> CSVには含まれていませんが、Snowflakeが自動で値を入れてくれます。
>
> **やらないと？** 「このデータはいつ、どのファイルから来たか」が追えなくなります。
> **やると？** データの出所を追跡でき、エラーや不整合の原因調査が容易になります。

```sql
-- Bronze層スキーマをデフォルトに設定
USE SCHEMA SENSOR_LAB.BRONZE;

-- Bronze層のテーブルを作成
-- OR REPLACE: 同名テーブルが既にあれば上書き作成（学習時は便利・本番は注意）
CREATE OR REPLACE TABLE RAW_SENSOR (
    timestamp       TIMESTAMP_NTZ,       -- タイムゾーンなし時刻（NTZ = No TimeZone）
                                         -- UTCで統一管理するため TZ なしを選択
    sensor_id       VARCHAR(10),         -- センサー識別子（例: S001）最大10文字
    location        VARCHAR(50),         -- 拠点名（例: tokyo）最大50文字
    temperature_c   FLOAT,               -- 気温（℃）小数を扱うのでFLOAT
    humidity_pct    FLOAT,               -- 湿度（%）同上
    battery_pct     FLOAT,               -- バッテリー残量（%）同上
    status          VARCHAR(10),         -- センサー状態（OK または ERROR）
    _loaded_at      TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
                                         -- ロード日時（DEFAULT で自動セット）
    _source_file    VARCHAR(255)         -- ロード元ファイル名（COPY INTO時にセット）
);

-- テーブルの構造（カラム名・型・NULL許可など）を確認
DESCRIBE TABLE RAW_SENSOR;
```

---

## 1-4 Snowsight でのデータプレビュー

> 💡 **GUIでのデータ確認の重要性**
> SQLを書く前にデータの中身を目で確認することで、
> カラムの型・NULL・想定外の値など問題を早期発見できます。
>
> **やらないと？** 型の不一致や想定外のデータに気づかず、後でエラーが出ます。
> **やると？** データの実態を把握してから加工処理を書けるので手戻りが減ります。

- 左メニュー「Data」→「Databases」→「SENSOR_LAB」→「BRONZE」→「RAW_SENSOR」
- 「Preview」タブでデータ確認（最大100行を素早く確認）
- 「Columns」タブでスキーマ（カラム名・型・NULL率）確認

---

## チェックリスト

- [ ] Database / Schema / Warehouse / Role / Stage の概念を説明できる
- [ ] Warehouseが課金の単位であることを理解した
- [ ] SnowsightのワークシートでSHOW / SELECTを実行できた
- [ ] RAW_SENSORテーブルを作成できた
- [ ] `_loaded_at` と `_source_file` カラムの役割を説明できる
