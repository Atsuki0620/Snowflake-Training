# Chapter 7：Task による定期自動実行

**目安時間**: 60〜90分
**ゴール**: ストアドプロシージャをTaskに登録し定期実行できるようにする

← [Chapter 6](06_ストアドプロシージャ.md) | [目次](README.md) | [Chapter 8 →](08_Python_APIs.md)

---

## このChapterで何をするか

前のChapterで作ったパイプラインを「自動化」します。
手動で `CALL SP_FULL_PIPELINE()` を実行しなくても、
Snowflakeのスケジューラが決まった時間に自動で実行してくれます。

> 💡 **Taskとは？**
> Snowflakeの組み込みスケジューラです。
> 「毎時0分にこのプロシージャを実行する」という設定をSnowflake内に登録でき、
> 外部のスケジューラ（cronサーバー・Airflowなど）を用意しなくても
> 定期実行を実現できます。
>
> **やらないと？** 毎日手動で `CALL SP_FULL_PIPELINE()` を実行する必要があります。
> **やると？** 設定したスケジュールで自動実行され、人手が不要になります。

---

## 7-1 Taskの基本（シンプル版）

> 💡 **まず1つのTaskで全処理を実行するシンプルな構成**
> 後述のTask Treeに進む前に、まず「1つのTaskで全パイプラインを動かす」
> シンプルな構成を試してみましょう。

```sql
-- Task作成: 毎時0分に SP_FULL_PIPELINE を実行
CREATE OR REPLACE TASK SENSOR_LAB.BRONZE.TASK_FULL_PIPELINE
    WAREHOUSE = COMPUTE_WH            -- このTaskの実行に使うウェアハウス
    SCHEDULE = 'USING CRON 0 * * * * Asia/Tokyo'
    -- CRON書式: 分 時 日 月 曜日 タイムゾーン
    -- 0 * * * *  = 毎時0分（*はワイルドカード = すべて）
    -- 別の例: '5 MINUTE' = 5分ごと / 'USING CRON 0 9 * * 1-5 Asia/Tokyo' = 平日9時
AS
    CALL SENSOR_LAB.BRONZE.SP_FULL_PIPELINE();  -- 実行するSQL（1文のみ指定可能）
```

> 💡 **CRONの読み方**
> CRON式は「スケジュールの設定言語」で、5つのフィールドで時間を表します：
> ```
> 分(0-59)  時(0-23)  日(1-31)  月(1-12)  曜日(0-6, 0=日曜)
>    0          *         *         *         *
>    ↑毎時0分    ↑毎時間   ↑毎日     ↑毎月    ↑曜日は問わない
> ```

---

## 7-2 Task Tree（依存関係を持つTask）

> 💡 **Task Treeとは？**
> 複数のTaskを「親→子→孫」の依存関係でつなぐ構成です。
> 親Taskが完了したら子Taskが自動で起動し、
> 子が完了したら孫が起動するという連鎖実行ができます。
>
> 1つのTaskに複数の処理をまとめる（SP_FULL_PIPELINE）方法と比べて、
> Task Treeにする利点は「どのステップで失敗したか」がTask単位で
> 明確にわかることです。
>
> **やらないと？** 全処理を1Taskにまとめると「Bronze失敗? Silver失敗?」の
> 切り分けが難しくなります。
> **やると？** Task Historyで各ステップの成否が個別に確認できます。

```sql
-- 【親Task】: Bronze ロード（スケジュールはここだけに設定）
CREATE OR REPLACE TASK SENSOR_LAB.BRONZE.TASK_BRONZE
    WAREHOUSE = COMPUTE_WH
    SCHEDULE = 'USING CRON 0 * * * * Asia/Tokyo'  -- 毎時0分に起動
AS
    CALL SENSOR_LAB.BRONZE.SP_LOAD_FROM_STAGE('BRONZE_STAGE', '.*sensor_.*\\.csv');

-- 【子Task】: Silver 変換（親Taskが完了したら自動で実行）
-- ※ AFTER で依存関係を持つTaskは同じスキーマに配置する必要があるため BRONZE スキーマに統一
CREATE OR REPLACE TASK SENSOR_LAB.BRONZE.TASK_SILVER
    WAREHOUSE = COMPUTE_WH
    AFTER SENSOR_LAB.BRONZE.TASK_BRONZE  -- TASK_BRONZE 完了後に実行
AS
    CALL SENSOR_LAB.SILVER.SP_TRANSFORM_TO_SILVER();

-- 【孫Task】: Gold 集計（子Taskが完了したら自動で実行）
CREATE OR REPLACE TASK SENSOR_LAB.BRONZE.TASK_GOLD
    WAREHOUSE = COMPUTE_WH
    AFTER SENSOR_LAB.BRONZE.TASK_SILVER  -- TASK_SILVER 完了後に実行
AS
    CALL SENSOR_LAB.GOLD.SP_AGGREGATE_TO_GOLD();
```

---

## 7-3 Task の有効化・無効化

> 💡 **⚠️ 子Taskから先にRESUMEしないといけない理由（重要な罠）**
> Task Treeでは「親を有効化すると即座にスケジュール実行が始まる」ため、
> 子が準備できていない状態で親を有効化すると、
> 子Taskが「SUSPENDED（停止中）」のまま連鎖実行に失敗します。
>
> **正しい順番: 孫 → 子 → 親 の順にRESUME**
>
> **やらないと？** 親がスケジュール実行されても、子・孫がSUSPENDEDのままで
> Silver・Goldが更新されません。
> **やると？** Task Treeが正常に機能し、Bronze→Silver→Goldが順番に自動実行されます。

```sql
-- ⚠️ 必ず子Taskから先に有効化すること（孫→子→親の順）
ALTER TASK SENSOR_LAB.BRONZE.TASK_GOLD   RESUME;   -- ① 孫を先に有効化
ALTER TASK SENSOR_LAB.BRONZE.TASK_SILVER RESUME;   -- ② 子を有効化
ALTER TASK SENSOR_LAB.BRONZE.TASK_BRONZE RESUME;   -- ③ 最後に親を有効化（ここでスケジュール開始）

-- Taskの状態を確認（STATE: started / suspended / running）
SHOW TASKS IN DATABASE SENSOR_LAB;

-- Taskを停止する場合（親Taskを停止すれば子・孫も連鎖が止まる）
ALTER TASK SENSOR_LAB.BRONZE.TASK_BRONZE SUSPEND;

-- 即時手動実行（スケジュールを待たずにテスト実行したい場合）
EXECUTE TASK SENSOR_LAB.BRONZE.TASK_BRONZE;
```

---

## 7-4 Task 実行履歴の確認

> 💡 **なぜ実行履歴を確認するのか？**
> Taskはバックグラウンドで自動実行されるため、
> 「ちゃんと動いているか」「エラーが出ていないか」を能動的に確認する必要があります。
> 定期的に確認する習慣をつけましょう。

```sql
-- 直近24時間のTask実行履歴を確認
-- STATE: succeeded / failed / skipped が確認できる
SELECT *
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
    TASK_NAME => 'TASK_BRONZE',
    SCHEDULED_TIME_RANGE_START => DATEADD(HOURS, -24, CURRENT_TIMESTAMP())  -- 24時間前から
))
ORDER BY SCHEDULED_TIME DESC;  -- 最新の実行から降順で表示
```

**Snowsight GUI で確認する方法**:
1. 左メニュー「Monitoring」→「Task History」
2. タスク名・状態（成功/失敗/スキップ）・実行時間・エラーをGUIで確認
3. 「Task Graph」タブでTask Treeの依存関係図が視覚的に確認できる

---

## チェックリスト

- [ ] Task Treeを作成できた（Bronze→Silver→Gold）
- [ ] CRON式の読み方を説明できる（`0 * * * *` = 毎時0分）
- [ ] 子Taskから先にRESUMEする理由を説明できる
- [ ] TaskをResume/Suspendできた
- [ ] EXECUTE TASKで即時手動実行できた
- [ ] TASK_HISTORYで実行結果を確認できた
- [ ] SnowsightのTask Historyで実行状況を確認できた
