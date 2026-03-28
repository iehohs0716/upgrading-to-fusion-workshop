# dbt Fusion マイグレーション トラブルシューティング記録

日付: 2026-03-28

---

## 1. dbt0432: PIVOT(ANY) の静的解析エラー

- **対象**: `models/marts/agg/monthly_payment_analysis_pivot_any.sql`
- **原因**: `PIVOT ... IN (ANY)` は動的SQLであり、dbt Fusionの静的解析が処理できない
- **対処法**:
  - `config` に `static_analysis='off'` を追加（適用済み）
  - または `IN (ANY)` を明示的な値リスト `IN ('credit_card', 'debit_card', 'paypal')` に変更

---

## 2. dbt1307: Snowflake の CHECK 制約未サポート

- **対象**: `models/marts/finance/_mart_finance__models.yml` / `financial_reporting_protected.sql`
- **原因**: Snowflakeは DDL での `CHECK` 制約をサポートしていない。`contract: enforced: true` が設定されていると、dbt が DDL に CHECK を含めるためエラーになる
- **ポイント**:
  - `contract: enforced: true` がないモデルでは `type: check` を書いても DDL に含まれないのでエラーにならない
  - SQL の `config()` と YAML の config が両方ある場合、**SQL が勝つ**（YAML だけ変えても SQL で上書きされる）
  - `customer_analytics_with_contracts` は YAML で `enforced: true` だが、SQL 側で `enforced: false` に上書きしていたためエラーにならなかった
- **対処法**: `contract: enforced: false` に変更、または CHECK 制約を削除

---

## 3. dbt1501: dbt_constraints パッケージの Fusion 互換性

- **原因**: `dbt_constraints` v1.0.7 が dbt Fusion の内部オブジェクト構造と互換性がなく、`none has no method named get` エラーが発生
- **対処法**: v1.0.8 にアップデート（Fusion 互換性が追加済み）
  - FK 制約はスキップされる（警告表示）、PK/UK 制約は正常動作

### dbt ネイティブ constraints vs dbt_constraints パッケージ

| | dbt ネイティブ | dbt_constraints パッケージ |
|---|---|---|
| タイミング | `CREATE TABLE` 時の DDL | `on-run-end` で `ALTER TABLE` |
| 根拠 | 無条件に付与 | テストがパスした場合のみ |
| CHECK | 対応（DWH 次第） | 非対応 |

---

## 4. dbt1005: Behavior Change Flags の扱い

- **原因**: dbt Fusion ではすべての behavior change flag が削除され、新しい動作が常に有効
- **対処法**: `dbt_project.yml` の `flags` セクションから不要なフラグを削除
- **移行戦略**: Fusion に切り替える前に、dbt Core 側で全フラグを `true` にしてテストしておくとサプライズを避けられる

---

## 5. config.meta_get() の互換性

- **対象**: `macros/add_custom_columns.sql`
- **原因**: `config.meta_get()` は dbt Fusion（Python モデル）で使えるメソッドだが、dbt Core の SQL モデル（Jinja コンテキスト）では存在しない
- **対処法**: `config.get('meta', {}).get('key')` を使えば両方で動く可能性があるが、Fusion での動作は要検証

---

## 6. dispatch 設定について

- マクロの検索・切り替え順序を制御する仕組み
- `search_order` でパッケージのマクロを自分のプロジェクトで上書き可能
- パッケージの優先度 > アダプターの優先度
- 今回のケースでは `dbt_constraints` をアップデートしたため不要

---

## 7. dbt の基礎知識メモ

### dbt seed
- CSV ファイルを DWH のテーブルとしてロードする機能
- マスタデータやテスト用データ向け。大量データには不向き

### dbt build vs dbt run
- `dbt run`: モデルのみ実行
- `dbt build`: モデル + テスト + seed + snapshot をすべて実行
- snapshot だけ実行: `dbt build --select "resource_type:snapshot"` または `dbt snapshot`

### materialized オプション
| 値 | 説明 | 用途 |
|---|---|---|
| `view` | ビュー、毎回計算 | ステージング層 |
| `table` | テーブル、実体化 | マート層（BI ツール等から直接参照） |
| `incremental` | 差分更新 | 大量データ、イベントログ |
| `ephemeral` | CTE として埋め込み | 中間ロジック |

- マート層を `table` にする理由: エンドユーザーが直接クエリするため、事前計算済みの方が高速

### sql_header
- `CREATE TABLE AS` / `CREATE VIEW AS` の前に差し込まれる SQL
- 用途: セッションパラメータ設定、一時 UDF 定義
- dbt Core v1.12 から generic data test でも使用可能（要フラグ有効化）
- Fusion ではフラグなしで常に使用可能

### パッケージの Fusion 対応確認方法
- GitHub のリリースノート / CHANGELOG を確認
- GitHub Issues で `fusion` を検索
- 実際に動かしてエラーを確認するのが最も確実
