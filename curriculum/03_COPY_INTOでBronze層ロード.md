# Chapter 3：COPY INTO でBronze層ロード

**目安時間**: 60〜90分
**ゴール**: ステージのCSVをテーブルへロードし、Bronze層を完成させる

← [Chapter 2](02_ステージとPUT.md) | [目次](README.md) | [Chapter 4 →](04_Silver層クレンジングと変換.md)

---

## このチャプターで何をするか

前のChapterでステージに預けたCSVを、実際にテーブルに取り込みます。

```
ステージ（一時置き場） ──COPY INTO──→ テーブル（RAW_SENSOR）
```

> 💡 **COPY INTOとは？**
> ステージ上のファイルをSnowflakeのテーブルに一括ロードするコマンドです。
> 単純なSQLのINSERTと違い、大量データの高速ロード・エラー制御・
> 重複ロード防止などの機能が組み込まれています。
>
> **やらないと？** ステージにファイルがあってもテーブルにデータが入りません。
> **やると？** CSVのデータがテーブルに格納され、SQLで分析できる状態になります。

---

## 3-1 基本的な COPY INTO

> 💡 **`METADATA$FILENAME` とは？**
> COPY INTO実行時にSnowflakeが自動で提供する「特殊な列」で、
> 現在ロード中のファイル名が入ります。
> これを `_source_file` カラムに記録することで、
> 「このデータはどのファイルから来たか」を後から追跡できます。
>
> **やらないと？** データの出所がわからなくなり、エラー時の原因調査が困難になります。
> **やると？** 「このNULLはどのファイルに起因するか」が一目でわかります。

> 💡 **`ON_ERROR = 'CONTINUE'` とは？**
> ロード中にエラー行（型の不一致など）が出たとき、エラーをスキップして
> 処理を続行するオプションです。
>
> | 設定値 | 動作 |
> |---|---|
> | `CONTINUE` | エラー行をスキップして続行（エラー行は無視） |
> | `SKIP_FILE` | エラーが出たファイル全体をスキップ |
> | `ABORT_STATEMENT`（デフォルト） | 1件でもエラーがあればロード全体を中断 |
>
> **やらないと？** デフォルトは `ABORT_STATEMENT` のため、1行でも型が合わないと
> ファイル全体がロードされません。
> **やると？** 多少汚いデータでもロードが完了し、エラー行だけ後から調査できます。

```sql
USE SCHEMA SENSOR_LAB.BRONZE;

-- 1ファイルをロード
COPY INTO RAW_SENSOR (
    -- ロード先カラムを明示（CSVの列順とテーブルのカラム順が一致しない場合も安全）
    timestamp, sensor_id, location, temperature_c, humidity_pct, battery_pct, status, _source_file
)
FROM (
    SELECT
        $1,                  -- CSVの1列目 → timestamp
        $2,                  -- CSVの2列目 → sensor_id
        $3,                  -- CSVの3列目 → location
        $4,                  -- CSVの4列目 → temperature_c
        $5,                  -- CSVの5列目 → humidity_pct
        $6,                  -- CSVの6列目 → battery_pct
        $7,                  -- CSVの7列目 → status
        METADATA$FILENAME    -- ロード元ファイル名（Snowflakeが自動で提供）
    FROM @BRONZE_STAGE/sensor_tokyo_20240115.csv  -- ステージ上の特定ファイルを指定
)
FILE_FORMAT = (FORMAT_NAME = 'CSV_FORMAT')  -- Chapter 2で作成したFile Formatを使用
ON_ERROR = 'CONTINUE';                       -- エラー行はスキップして続行

-- ロード結果を確認
SELECT * FROM RAW_SENSOR LIMIT 10;
SELECT COUNT(*) FROM RAW_SENSOR;  -- 行数の確認（期待値: 432行）
```

---

## 3-2 全ファイルを一括ロード（パターンマッチ）

> 💡 **PATTERNオプションとは？**
> ステージ上のファイルを正規表現でフィルタリングして、
> 条件に合うファイルだけをまとめてロードできます。
> ファイルを1つずつ手動でロードする必要がなくなります。
>
> **やらないと？** ファイルが10個あれば10回COPY INTOを実行する必要があります。
> **やると？** 1回のCOPY INTOで全ファイルを一括ロードできます。

```sql
-- ステージ上の全CSVを一括ロード（PATTERNで対象を絞る）
COPY INTO RAW_SENSOR (timestamp, sensor_id, location, temperature_c, humidity_pct, battery_pct, status, _source_file)
FROM (
    SELECT $1, $2, $3, $4, $5, $6, $7, METADATA$FILENAME
    FROM @BRONZE_STAGE  -- ファイル名を指定せずステージ全体を参照
)
FILE_FORMAT = (FORMAT_NAME = 'CSV_FORMAT')
PATTERN = '.*sensor_.*\.csv'  -- 正規表現: "sensor_" を含む .csv ファイルのみ対象
ON_ERROR = 'CONTINUE';
```

---

## 3-3 ロードエラーの確認

> 💡 **なぜエラーを確認する必要があるのか？**
> `ON_ERROR = 'CONTINUE'` にするとエラー行はスキップされますが、
> スキップされたこと自体は通知されません。
> COPY_HISTORYを確認することで「何行スキップされたか」「どんなエラーか」がわかります。
>
> **やらないと？** データが欠損していても気づかず、後の分析結果が正確でなくなります。
> **やると？** ロードの品質を保証でき、問題があれば早期に対処できます。

```sql
-- 直近1時間以内のロード履歴を確認
-- ロードしたファイル名・行数・エラー数・状態が確認できる
SELECT *
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME => 'RAW_SENSOR',
    START_TIME => DATEADD(HOURS, -1, CURRENT_TIMESTAMP())  -- 1時間前から現在まで
));

-- エラー内容の詳細確認（COPY_HISTORYで取得したJOB_IDを使う）
SELECT *
FROM TABLE(VALIDATE(RAW_SENSOR, JOB_ID => '<JOB_ID>'));
```

---

## 3-4 ロード状況の確認（Snowsight GUI）

> 💡 **Query Historyで何がわかるか？**
> SnowsightのGUIを使うと、SQLを書かなくてもロード結果を確認できます。
> 特にエラーが発生した場合、エラーメッセージが視覚的に表示されるため
> 原因特定が容易です。

- 左メニュー「Monitoring」→「Query History」でCOPY INTOの実行履歴を確認
- 実行時間・処理行数・スキップ行数・エラー内容をGUIで確認できる
- 特定のクエリをクリックすると実行計画（Query Profile）も確認できる

---

## 3-5 重複ロード防止（べき等性）

> 💡 **Snowflakeの重複ロード防止機能とは？**
> Snowflakeは一度ロードしたファイルのメタデータ（ファイル名・サイズ・更新日時）を
> 内部で記憶しています。同じファイルを再度COPY INTOしようとすると
> 自動でスキップされます。
>
> これを「**べき等性（Idempotency）**」といいます。
> 何度実行しても同じ結果になるため、エラーで再実行してもデータが2重になりません。
>
> **やらないと？**（この機能がなければ）同じファイルを誤って2回ロードすると
> データが2倍になります。
> **やると？** 安心して再実行できます（失敗したバッチをそのまま再実行してOK）。

```sql
-- Snowflakeは同一ファイルの重複ロードをデフォルトでスキップする
-- 試しに同じCOPY INTOをもう一度実行してみると "0 rows loaded" になる

-- ⚠️ FORCE = TRUE は「重複チェックをスキップして強制ロード」するオプション
-- テスト目的以外では使わないこと（データが2重になります）
COPY INTO RAW_SENSOR -- ...（同じCOPY INTO文）
FORCE = TRUE;  -- 重複チェックをバイパスして再ロード（テスト・リセット時のみ使用）
```

---

## チェックリスト

- [ ] COPY INTO で1ファイルをロードできた
- [ ] `METADATA$FILENAME` がテーブルに記録されていることを確認した
- [ ] PATTERNで全ファイルを一括ロードできた
- [ ] COPY_HISTORYでロード履歴（行数・エラー数）を確認できた
- [ ] 同じファイルを再度COPY INTOしてもスキップされることを確認した（べき等性）
