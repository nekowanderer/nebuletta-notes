# Apache Kafka、Apache Hudi、AWS Athena を使用したリアルタイムデータクエリの実践

[English](../en/11_analysis_pipeline_for_athena_practice.md) | [繁體中文](../zh-tw/11_analysis_pipeline_for_athena_practice.md) | [日本語](./11_analysis_pipeline_for_athena_practice.md) | [インデックスに戻る](../README.md)

---

## ケース背景

現代のマイクロサービスアーキテクチャでは、各マイクロサービスがデータベースに書き込む際に、データをデータレイクに同期させ、後続の分析担当者が AWS Athena でクエリを実行できるようにすることが一般的な要件となっています。本ケーススタディでは、Apache Kafka、Apache Hudi、AWS Athena を統合して、効率的でスケーラブルなリアルタイムデータクエリパイプラインを実装する方法を紹介します。

## アーキテクチャ概要

1. **マイクロサービスデータ書き込み**
   - 各マイクロサービスがそれぞれのデータベース（Oracle、DynamoDB など）にデータを書き込みます。
2. **変更データキャプチャ（CDC）**
   - CDC ツール（Oracle GoldenGate、Debezium、DynamoDB Streams など）を使用してデータ変更イベントをキャプチャし、Apache Kafka にプッシュします。
3. **Apache Kafka イベントストリーム**
   - Kafka はイベントブローカーとして機能し、すべてのデータ変更イベントを受信して配信します。
4. **データ処理と Apache Hudi 書き込み**
   - AWS Lambda または Kafka Connect を使用して Kafka イベントを処理し、Apache Hudi（AWS S3 に保存）に変換して書き込みます。
5. **AWS Athena クエリ**
   - Athena は S3 上の Hudi データをクエリし、SQL 分析とレポート作成をサポートします。

## 詳細な手順

### 1. 変更データキャプチャ（CDC）
- **Oracle**: Oracle GoldenGate または Debezium を使用してデータ変更をキャプチャし、Kafka にプッシュします。
- **DynamoDB**: DynamoDB Streams を有効にして、テーブル変更イベントをキャプチャします。

### 2. Kafka ブローカーレイヤーの設計
- CDC ツールからのイベントを受信する Kafka クラスターをデプロイします。
- 各データソースに対応する Kafka トピックを設計します。

### 3. データ処理と Apache Hudi 書き込み
- **AWS Lambda 処理**: Lambda が Kafka トピックを監視し、イベントを Hudi 形式に変換して S3 に書き込みます。
- **Kafka Connect 処理**: Hudi Sink Connector をデプロイして、Kafka トピックデータを直接 Hudi テーブルに書き込みます（高スループット要件に適しています）。

### 4. AWS Athena による Hudi データのクエリ
- AWS Glue Data Catalog に Hudi テーブル構造を登録します。
- Athena を使用して S3 上の Hudi テーブルをクエリし、SQL 分析をサポートします。

## まとめと実践的な推奨事項

このアーキテクチャは、データ変更のリアルタイムキャプチャ、処理、保存、クエリを効果的に実装し、効率的なデータ処理と分析が必要なシナリオに適しています。推奨事項：
- データベースタイプと予算に基づいて CDC ツールを選択する
- データソースとパーティショニング戦略を考慮して Kafka トピックを設計する
- クエリパターンに基づいて適切なテーブルタイプ（Copy-on-write または Merge-on-read）を選択する
- Athena クエリの前に、Glue Catalog とデータパーティショニングを最適化する

---

このケーススタディは、企業がリアルタイムデータレイククエリを導入する際の参照アーキテクチャとして活用できます。
