# Chapter 4：Silver層 — クレンジング & 変換

**目安時間**: 90〜120分
**ゴール**: Bronze の生データをクレンジングし、Silver層を構築する

← [Chapter 3](03_COPY_INTOでBronze層ロード.md) | [目次](README.md) | [Chapter 5 →](05_Gold層集計と分析.md)

---

## このチャプターで何をするか

Bronzeに保存した「生データ（汚いまま）」を、分析で使える「きれいなデータ」に変換します。

```
Bronze（RAW_SENSOR）──変換SQL──→ Silver（SENSOR_CLEAN）
```

> 💡 **なぜBronzeのデータをそのまま分析に使わないのか？**
> 生データには NULL・外れ値・重複・型の不統一などが含まれることが多く、
> そのまま集計すると誤った結果になります。
> またBronzeは「元データを保全する層」なので、加工はSilverで行います。
>
> **やらないと？** NULLや外れ値が混じったまま平均を計算すると、
> 「異常値1件が全体の平均を大きく歪める」などの問題が起きます。
> **やると？** 信頼性の高いクリーンデータが手に入り、分析結果の品質が保証されます。

---

## 4-1 Silver層テーブルの作成

> 💡 **BronzeとSilverでカラム構成が変わる理由**
> Bronzeは「入ってきたデータをそのまま保存」するため、CSVの列がほぼそのまま入ります。
> Silverでは「分析しやすい形」に変換するため、カラムを追加・変換します。
>
> 新しく追加するカラム:
> - `recorded_date`: タイムスタンプから日付だけを抜き出す（日次集計で使う）
> - `recorded_hour`: 時間帯だけを抜き出す（時間帯別分析で使う）
> - `is_anomaly`: 外れ値かどうかのフラグ（削除せずフラグで管理する理由は後述）

```sql
USE SCHEMA SENSOR_LAB.SILVER;

CREATE OR REPLACE TABLE SENSOR_CLEAN (
    sensor_id       VARCHAR(10),         -- センサー識別子（Bronze から引き継ぎ）
    location        VARCHAR(50),         -- 拠点名（Bronze から引き継ぎ）
    recorded_at     TIMESTAMP_NTZ,       -- 記録日時（Bronze の timestamp をリネーム）
    recorded_date   DATE,                -- 記録日（timestamp から DATE 部分のみ抽出）
    recorded_hour   INT,                 -- 記録時間（0〜23 の整数、時間帯分析用）
    temperature_c   FLOAT,               -- 気温（クレンジング済み・外れ値はNULL）
    humidity_pct    FLOAT,               -- 湿度（クレンジング済み・範囲外は境界値に補正）
    battery_pct     FLOAT,               -- バッテリー残量（そのまま引き継ぎ）
    status          VARCHAR(10),         -- センサー状態
    is_anomaly      BOOLEAN,             -- 外れ値フラグ（TRUE = 異常値が含まれていた行）
    _loaded_at      TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),  -- Silver へのロード日時
    _source_file    VARCHAR(255)         -- 元ファイル名（Bronze から引き継ぎ）
);
```

---

## 4-2 Bronze → Silver 変換SQL（クレンジングロジック）

> 💡 **外れ値を「削除」せず「NULLに変換」する理由**
> 異常な値を単純に削除すると、「元々その時間帯にデータが存在したこと」自体が
> 失われてしまいます。NULLに変換することで「データは存在するが値が無効」という
> 事実を残せます。`is_anomaly` フラグも同様に、「異常があった」という記録を保持します。
>
> **やらないと？** 「センサーが壊れていたのか」「データがなかったのか」が区別できません。
> **やると？** 分析時に「異常件数が多い拠点」を特定するなどの活用ができます。

> 💡 **`QUALIFY ROW_NUMBER()` で重複を排除する理由**
> 同じCSVを誤って2回ロードした場合など、Bronze層に同一センサー・同一時刻の
> データが複数入ることがあります。
> `QUALIFY` でウィンドウ関数の結果を絞ることで、最新のロードデータだけを残せます。
>
> **やらないと？** 重複データがSilverに入り、集計結果が2倍になるなどの問題が起きます。
> **やると？** 「センサーID + タイムスタンプ」の組み合わせで必ず1件だけ保証されます。

```sql
INSERT INTO SENSOR_LAB.SILVER.SENSOR_CLEAN
SELECT
    sensor_id,
    location,
    timestamp                               AS recorded_at,   -- 列名をリネーム
    DATE(timestamp)                         AS recorded_date,  -- タイムスタンプから日付部分を抽出
    HOUR(timestamp)                         AS recorded_hour,  -- タイムスタンプから時間（0〜23）を抽出

    -- 気温の外れ値処理: センサー物理的限界(-50℃〜80℃)を超える値はNULLに
    CASE
        WHEN temperature_c IS NULL     THEN NULL  -- 元々NULLはそのままNULL
        WHEN temperature_c < -50       THEN NULL  -- -50℃未満はセンサー異常→NULL
        WHEN temperature_c > 80        THEN NULL  -- 80℃超はセンサー異常→NULL
        ELSE temperature_c                        -- 正常範囲はそのまま
    END                                     AS temperature_c,

    -- 湿度の補正: 物理的にありえない値（0%未満・100%超）は境界値にクランプ
    CASE
        WHEN humidity_pct IS NULL      THEN NULL  -- 元々NULLはそのままNULL
        WHEN humidity_pct < 0          THEN 0     -- 0%未満は0%に丸める
        WHEN humidity_pct > 100        THEN 100   -- 100%超は100%に丸める
        ELSE humidity_pct                         -- 正常範囲はそのまま
    END                                     AS humidity_pct,

    battery_pct,  -- バッテリーは補正なしでそのまま引き継ぎ
    status,

    -- 外れ値フラグ: 気温が異常範囲だった場合にTRUE（値をNULLにしても記録は残す）
    (temperature_c < -50 OR temperature_c > 80)  AS is_anomaly,

    CURRENT_TIMESTAMP()                     AS _loaded_at,   -- Silverロード日時
    _source_file                                             -- 元ファイル名を引き継ぎ
FROM SENSOR_LAB.BRONZE.RAW_SENSOR

-- 重複排除: センサーID + タイムスタンプが同じ行が複数あれば、最新ロード分のみ残す
-- QUALIFY はウィンドウ関数の結果でフィルタをかけるSnowflake/BigQuery特有の構文
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY sensor_id, timestamp  -- この組み合わせでグループ化
    ORDER BY _loaded_at DESC           -- 最新ロード順に並べて
) = 1;                                 -- 各グループの1件目（最新）だけを取得
```

---

## 4-3 クレンジング結果の確認

> 💡 **クレンジング後の品質チェックが重要な理由**
> クレンジングSQLが正しく動いていても、想定より多くのNULLが出るなど
> データ品質の問題を早期に発見できます。

```sql
-- NULL件数の確認（何件がクレンジングで除外されたかを把握）
SELECT
    COUNT(*)                                    AS total,         -- 総件数
    SUM(CASE WHEN temperature_c IS NULL THEN 1 ELSE 0 END) AS temp_nulls,     -- 気温NULL件数
    SUM(CASE WHEN humidity_pct IS NULL THEN 1 ELSE 0 END)  AS hum_nulls,      -- 湿度NULL件数
    SUM(CASE WHEN is_anomaly THEN 1 ELSE 0 END)            AS anomalies        -- 外れ値件数
FROM SENSOR_LAB.SILVER.SENSOR_CLEAN;

-- 拠点・日付ごとのデータ件数確認（拠点によって件数が大きく違う場合は要調査）
SELECT location, recorded_date, COUNT(*) AS cnt
FROM SENSOR_LAB.SILVER.SENSOR_CLEAN
GROUP BY location, recorded_date
ORDER BY location, recorded_date;
```

---

## 4-4 VIEWの作成（正常データのみ）

> 💡 **VIEWとは？**
> 「よく使うSELECTクエリに名前をつけたもの」です。
> テーブルのようにデータを保持するのではなく、参照するたびにSQLが実行されます。
>
> ここでは「分析で使う正常データのみ」を返すビューを作ります。
> 以降のChapterでは `SENSOR_CLEAN` テーブルではなくこのビューを使うことで、
> 異常値・NULLを含まないデータだけを分析対象にできます。
>
> **やらないと？** 集計SQLごとに `WHERE temperature_c IS NOT NULL AND ...` を
> 毎回書く必要があり、書き忘れると正確でない集計になります。
> **やると？** フィルタ条件を一元管理でき、集計SQLがシンプルになります。

```sql
-- 正常データのみを返すビューを作成
-- このビューを使えば常にクリーンなデータだけを参照できる
CREATE OR REPLACE VIEW SENSOR_LAB.SILVER.V_SENSOR_VALID AS
SELECT *
FROM SENSOR_LAB.SILVER.SENSOR_CLEAN
WHERE temperature_c IS NOT NULL   -- 気温がNULLでない（クレンジングで除外されていない）
  AND humidity_pct IS NOT NULL    -- 湿度がNULLでない
  AND status = 'OK'               -- センサー状態が正常
  AND NOT is_anomaly;             -- 外れ値フラグが立っていない
```

---

## チェックリスト

- [ ] SENSOR_CLEANテーブルを作成できた
- [ ] クレンジングSQLを実行してBronze → Silverのデータが入った
- [ ] NULL件数・外れ値件数を確認した
- [ ] `QUALIFY ROW_NUMBER()` による重複排除の仕組みを説明できる
- [ ] V_SENSOR_VALID ビューを作成できた
- [ ] なぜ外れ値を削除せずNULLに変換するのかを説明できる
