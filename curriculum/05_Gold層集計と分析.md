# Chapter 5：Gold層 — 集計 & 分析

**目安時間**: 60〜90分
**ゴール**: Silver層から集計テーブルを作りビジネス価値のあるデータを整備する

← [Chapter 4](04_Silver層クレンジングと変換.md) | [目次](README.md) | [Chapter 6 →](06_ストアドプロシージャ.md)

---

## このチャプターで何をするか

Silverのクリーンデータを集計・要約し、ダッシュボードや分析に使いやすい形にします。

```
Silver（V_SENSOR_VALID）──集計SQL──→ Gold（HOURLY_STATS / DAILY_STATS）
```

> 💡 **なぜGold層が必要なのか？SilverのVIEWを使えばいいのでは？**
> ビュー経由で毎回集計すると、参照のたびに全データをスキャンして計算するため
> レポートや大量アクセスがあると遅くなります。
> Gold層に集計済みの結果を「事前計算」して保存しておくことで
> 高速・低コストで参照できます。
>
> | | ビューで毎回集計 | Gold層に保存 |
> |---|---|---|
> | 速度 | 遅い（毎回フルスキャン） | 速い（集計済み） |
> | コスト | 毎回ウェアハウス課金 | 参照は安い |
> | 鮮度 | 常に最新 | 更新タイミングによる |
>
> **やらないと？** ダッシュボードが開くたびに数秒〜数十秒待たされます。
> **やると？** 集計結果が即座に返り、ユーザー体験が向上します。

---

## 5-1 Gold層テーブルの設計

> 💡 **なぜ「時間ごと」と「日次」の2つのテーブルを作るのか？**
> 用途によって必要な粒度が異なります。
> 時間帯別の傾向を見たいなら `HOURLY_STATS`、日次サマリーなら `DAILY_STATS` を使います。
> また、`DAILY_STATS` は `HOURLY_STATS` を集計したものなので、
> 計算の二重化を避けるためにこの2段構造にしています。

```sql
USE SCHEMA SENSOR_LAB.GOLD;

-- 1時間ごとの集計テーブル（拠点 × 日付 × 時間帯の粒度）
CREATE OR REPLACE TABLE HOURLY_STATS (
    location        VARCHAR(50),  -- 拠点名
    stat_date       DATE,         -- 集計対象日
    stat_hour       INT,          -- 集計対象時間（0〜23）
    sensor_count    INT,          -- その時間帯に稼働していたセンサーのユニーク数
    avg_temp        FLOAT,        -- 平均気温（小数点2桁に丸め済み）
    min_temp        FLOAT,        -- 最低気温
    max_temp        FLOAT,        -- 最高気温
    avg_humidity    FLOAT,        -- 平均湿度
    min_humidity    FLOAT,        -- 最低湿度
    max_humidity    FLOAT,        -- 最高湿度
    anomaly_count   INT,          -- 異常値が検出された件数
    _loaded_at      TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()  -- Gold へのロード日時
);

-- 日次の集計テーブル（拠点 × 日付の粒度）
CREATE OR REPLACE TABLE DAILY_STATS (
    location        VARCHAR(50),  -- 拠点名
    stat_date       DATE,         -- 集計対象日
    sensor_count    INT,          -- その日に稼働していたセンサー数（HOURLY の合計）
    avg_temp        FLOAT,        -- 日平均気温
    min_temp        FLOAT,        -- 日最低気温
    max_temp        FLOAT,        -- 日最高気温
    avg_humidity    FLOAT,        -- 日平均湿度
    min_humidity    FLOAT,        -- 日最低湿度
    max_humidity    FLOAT,        -- 日最高湿度
    anomaly_count   INT,          -- 日合計異常件数
    _loaded_at      TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);
```

---

## 5-2 Silver → Gold 集計SQL

> 💡 **`COUNT(DISTINCT sensor_id)` とは？**
> 単純な `COUNT(*)` は「データ行数」を数えますが、
> `COUNT(DISTINCT sensor_id)` は「ユニークなセンサーの数」を数えます。
> 1つのセンサーが1時間に複数回データを送る場合、
> センサーの「稼働台数」を知りたいときは DISTINCT が必要です。
>
> **やらないと？** データ行数 = センサー台数ではないため、稼働台数が正確に取れません。

```sql
-- 1時間ごとの集計（V_SENSOR_VALID ビューを使うことで異常値・NULLを自動除外）
INSERT INTO SENSOR_LAB.GOLD.HOURLY_STATS
SELECT
    location,
    recorded_date               AS stat_date,
    recorded_hour               AS stat_hour,
    COUNT(DISTINCT sensor_id)   AS sensor_count,         -- ユニークセンサー数
    ROUND(AVG(temperature_c),2) AS avg_temp,             -- 平均気温（小数点2桁）
    MIN(temperature_c)          AS min_temp,             -- 最低気温
    MAX(temperature_c)          AS max_temp,             -- 最高気温
    ROUND(AVG(humidity_pct),2)  AS avg_humidity,         -- 平均湿度
    MIN(humidity_pct)           AS min_humidity,
    MAX(humidity_pct)           AS max_humidity,
    SUM(CASE WHEN is_anomaly THEN 1 ELSE 0 END) AS anomaly_count,  -- 異常件数の合計
    CURRENT_TIMESTAMP()         AS _loaded_at
FROM SENSOR_LAB.SILVER.V_SENSOR_VALID  -- NULLや異常値を除いたビューを使用
GROUP BY location, recorded_date, recorded_hour;  -- 拠点・日付・時間帯でグループ化

-- 日次集計（HOURLY_STATSを集計してDATEレベルに丸める）
INSERT INTO SENSOR_LAB.GOLD.DAILY_STATS
SELECT
    location,
    stat_date,
    SUM(sensor_count)           AS sensor_count,   -- 時間ごとのセンサー数を1日分合計
    ROUND(AVG(avg_temp),2)      AS avg_temp,        -- 時間帯平均の平均 = 日平均
    MIN(min_temp)               AS min_temp,        -- 全時間帯の最低気温
    MAX(max_temp)               AS max_temp,        -- 全時間帯の最高気温
    ROUND(AVG(avg_humidity),2)  AS avg_humidity,
    MIN(min_humidity)           AS min_humidity,
    MAX(max_humidity)           AS max_humidity,
    SUM(anomaly_count)          AS anomaly_count,   -- 全時間帯の異常件数を合計
    CURRENT_TIMESTAMP()         AS _loaded_at
FROM SENSOR_LAB.GOLD.HOURLY_STATS  -- SilverではなくHOURLYから集計（2重計算防止）
GROUP BY location, stat_date;
```

---

## 5-3 分析クエリ例

> 💡 **Gold層ができると何ができるか？**
> 集計済みのテーブルに対してSELECTするだけなので、
> 複雑なJOINや集計なしにシンプルなクエリで洞察が得られます。

```sql
-- 拠点別の日次平均気温ランキング（どの拠点が最も暑い日か）
SELECT location, stat_date, avg_temp
FROM SENSOR_LAB.GOLD.DAILY_STATS
ORDER BY stat_date, avg_temp DESC;  -- 日付ごとに気温の高い順に並べる

-- 異常検知が多い拠点TOP（設備トラブルが多い拠点を把握）
SELECT location, SUM(anomaly_count) AS total_anomalies
FROM SENSOR_LAB.GOLD.DAILY_STATS
GROUP BY location
ORDER BY total_anomalies DESC;  -- 異常件数が多い順
```

---

## 5-4 Snowsight でのチャート作成

> 💡 **SnowsightのChart機能とは？**
> クエリ結果をそのままグラフ化できる機能です。
> BIツール（Tableau・PowerBIなど）を使わなくても
> 簡単な可視化がSnowsight内で完結します。
>
> **やらないと？** 数値の羅列だけで、傾向が直感的に把握しづらいです。
> **やると？** 気温の推移・拠点間比較などが一目でわかるグラフになります。

1. 上記SQLをワークシートで実行（`stat_date`, `location`, `avg_temp` を含むクエリ）
2. 結果パネル右上「Chart」タブをクリック
3. グラフタイプ: 折れ線グラフ（Line）を選択
4. X軸: `STAT_DATE`、Y軸: `AVG_TEMP`、カラー分け: `LOCATION` を設定
5. 拠点別の気温推移が折れ線グラフで表示される

---

## チェックリスト

- [ ] HOURLY_STATS / DAILY_STATSテーブルを作成できた
- [ ] 集計SQLを実行してGold層にデータを格納できた
- [ ] `COUNT(DISTINCT sensor_id)` と `COUNT(*)` の違いを説明できる
- [ ] DAILY_STATSをHOURLY_STATSから作る理由を説明できる
- [ ] Snowsightのチャート機能で気温推移グラフを表示できた
