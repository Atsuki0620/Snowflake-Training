# Snowflake センサーデータ取込・自動化 習得カリキュラム

> **対象者**: Pythonは業務経験あり・Snowflakeはほぼ未経験
> **期間**: 1〜2週間（1日30分〜1時間）
> **スタイル**: ハンズオン中心・チャプター別ファイル形式
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

## なぜこのカリキュラムの順番なのか？

このカリキュラムは「生データを受け取り → きれいにして → 集計する」という
**データパイプラインの流れ**をそのまま順番に学ぶ構成になっています。

| フェーズ | 学ぶこと |
|---|---|
| Ch.0〜1 | 土台づくり（環境・概念） |
| Ch.2〜3 | データを「入れる」（取込） |
| Ch.4〜5 | データを「整える・活かす」（変換・集計） |
| Ch.6〜7 | 処理を「自動化する」（プロシージャ・タスク） |
| Ch.8 | Pythonから「管理する」（API操作） |

---

## チャプター一覧

| # | ファイル | 目安時間 | ゴール |
|---|---|---|---|
| 0 | [00_環境セットアップ.md](00_環境セットアップ.md) | 30〜60分 | ログイン・CLI・Pythonの接続確認 |
| 1 | [01_基本概念とSnowsight操作.md](01_基本概念とSnowsight操作.md) | 60〜90分 | 主要概念の理解・テーブル作成 |
| 2 | [02_ステージとPUT.md](02_ステージとPUT.md) | 60〜90分 | CSVをSnowflakeにアップロード |
| 3 | [03_COPY_INTOでBronze層ロード.md](03_COPY_INTOでBronze層ロード.md) | 60〜90分 | ステージからテーブルへロード |
| 4 | [04_Silver層クレンジングと変換.md](04_Silver層クレンジングと変換.md) | 90〜120分 | NULL・外れ値処理・Silver構築 |
| 5 | [05_Gold層集計と分析.md](05_Gold層集計と分析.md) | 60〜90分 | 集計テーブルの構築・可視化 |
| 6 | [06_ストアドプロシージャ.md](06_ストアドプロシージャ.md) | 90〜120分 | パイプラインの1コール実行 |
| 7 | [07_Taskによる定期自動実行.md](07_Taskによる定期自動実行.md) | 60〜90分 | スケジュール実行の設定 |
| 8 | [08_Python_APIs.md](08_Python_APIs.md) | 60〜90分 | PythonからSnowflakeオブジェクトを管理 |

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

**CSVスキーマ（各ファイル共通）**:
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

## 参考リンク

- [Snowflake ドキュメント（公式）](https://docs.snowflake.com/)
- [COPY INTO コマンドリファレンス](https://docs.snowflake.com/en/sql-reference/sql/copy-into-table)
- [Snowflake Connector for Python](https://docs.snowflake.com/en/developer-guide/python-connector/python-connector)
- [Snowflake Python APIs](https://docs.snowflake.com/en/developer-guide/snowflake-python-api/snowflake-python-api-reference)
- [Tasks and Task Graphs](https://docs.snowflake.com/en/user-guide/tasks-intro)
- [SnowSQL インストールガイド](https://docs.snowflake.com/en/user-guide/snowsql-install-config)
