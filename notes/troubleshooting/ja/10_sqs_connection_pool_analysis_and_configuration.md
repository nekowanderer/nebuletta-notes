# SQS コネクションプール分析と設定

[English](../en/10_sqs_connection_pool_analysis_and_configuration.md) | [繁體中文](../zh-tw/10_sqs_connection_pool_analysis_and_configuration.md) | [日本語](10_sqs_connection_pool_analysis_and_configuration.md) | [インデックスに戻る](../README.md)

---

## 背景
- 実験日: 2025/12/09
- 難易度：🤬🤬🤬
- 説明: Production 環境で4つ目の SQS queue を追加した後、コネクションプール枯渇エラーが発生しました。エラーメッセージは、設定された最大時間内にプールからコネクションを取得できなかったことを示しています。

## 発生した現象

- Kibana ログ：
  ```
  Unable to execute HTTP request: Acquire operation took longer than the configured maximum time.
  This indicates that a request cannot get a connection from the pool within the specified maximum time.
  ```
- エラーは `QueueAsyncClientImpl` が non-transactional EMAIL messages を処理する際に発生（下層で AWS SDK V2 を直接呼び出している）
- サービスが新しい queue からのメッセージを正常に処理できなかった

## デバッグプロセス

### 第一段階：インフラストラクチャ設定の確認

以下の項目が正しく設定されていることを確認しました：

| 項目 | 状態 | 説明 |
|------|------|------|
| SQS Queues | ✅ | 元々3つの queue があり、4つ目の queue を追加 |
| Poller 設定 | ✅ | 1秒ごとにメッセージを取得（`sqs.communication.poller.schedule: "1s"`）、これはアプリケーションで定義されたカスタムプロパティ |
| Batch Size | ✅ | Transactional: 10, Non-transactional: 10、これは毎回 SQS から取得するメッセージ数、max = 10 |
| SQS Client | ⚠️ | AWS SDK のデフォルト設定を使用（問題はここにある） |

### 第二段階：Queue の動作パターン分析

各 queue の動作パターンを確認しました：

**元々の3つの Queue：**
- `transactionalQueueUrl`: Transactional message queue
- `queueUrl`: Non-transactional SMS message queue  
- `emailMetricsQueueUrl`: Email metrics queue

**新しく追加した4つ目の Queue：**
- Non-transactional EMAIL message queue
- 各 poller 実行ごと：
  - 1回の `receiveMessage` 呼び出し（最大10メッセージをバッチ読み取り）
  - 10回の `deleteMessage` 呼び出し（メッセージごとに1回）
  - **合計：11個の HTTP コネクションが必要**

**結論**：新しい queue の動作パターンはコネクション集約型で、実行ごとに11個のコネクションが必要です。

### 第三段階：コネクションプール容量分析

SQS Client のコネクションプール設定を確認しました：

| 設定項目 | デフォルト値 | 実際の要件 |
|---------|-------------|-----------|
| `max-concurrency` | 50 | 不足 |
| `connection-acquisition-timeout-seconds` | 2秒 | 短すぎる |
| `max-pending-connection-acquires` | 10,000 | 十分 |

**発見事項**：
- デフォルトのコネクションプールは50個のコネクションのみ
- 新しい queue は毎秒11個のコネクションを必要とし、他の queue のトラフィックと合わせると、容易に50個を超える
- コネクション取得タイムアウトは2秒のみで、高トラフィックシナリオではコネクション解放を待つには不十分

## 根本原因

### 新しい Queue の追加によるコネクションプール枯渇

**重要な発見**：4つ目の queue を追加する際、この queue はコネクション集約型の動作パターン（実行ごとに1つの receiveMessage + 10個の deleteMessage = 11個のコネクションが必要）を採用し、並行 HTTP コネクション要件を大幅に増加させ、デフォルトのコネクションプール容量50を超過しました。

### アーキテクチャの変更

**変更前（3つの Queue）：**
- `transactionalQueueUrl`: Transactional message queue
- `queueUrl`: Non-transactional SMS message queue  
- `emailMetricsQueueUrl`: Email metrics queue
- 各 poller 実行でこれらの queue のメッセージを処理

**変更後（4つの Queue - 追加後）：**
- `transactionalQueueUrl`: Transactional message queue
- `queueUrl`: Non-transactional SMS message queue  
- `emailMetricsQueueUrl`: Email metrics queue
- **新しい queue**: Non-transactional EMAIL message queue（新規追加）
  - 各 poller 実行：
    - 1回の `receiveMessage` 呼び出し（最大10メッセージをバッチ読み取り）
    - 10回の `deleteMessage` 呼び出し（メッセージごとに1回）
    - **合計：11個の HTTP コネクションが必要**

### 影響

- **コネクション集約型パターン**：新しい queue は各 poller 実行ごとに11個のコネクションが必要（1つの receiveMessage + 10個の deleteMessage）
- **コネクションプール圧力の増加**：poller が1秒ごとに実行されるため、大量の並行コネクション要件が追加される
- **コネクションプール枯渇**：デフォルトのコネクションプール（50個のコネクション）は、他の queue のトラフィックと組み合わせると不十分
- **コネクション取得タイムアウト**：コネクションプールが枯渇すると、新しい API 呼び出しがタイムリーにコネクションを取得できない

## 問題の原理分析

### コネクションプールの動作メカニズム

1. **コネクションプールの位置**：
   - コネクションプールは**SQS サービス側ではなく**、**アプリケーション側**（アプリケーション内部）にある
   - `NettyNioAsyncHttpClient` によって管理される（Quarkus フレームワークを使用しているため）
   - アプリケーションメモリ内に存在
   - HTTP コネクションの再利用に使用される

2. **コネクションプールの動作フロー**：
   ```java
   // 呼び出すと
   sqsAsyncClient.receiveMessage(...)
   // SDK 内部では：
   // 1. コネクションプールから HTTP コネクションを取得
   // 2. このコネクションを使用して SQS に HTTP リクエストを送信
   // 3. レスポンスを受信
   // 4. コネクションをプールに返却（次回使用のため）
   ```

3. **なぜコネクションプールが必要か？**
   - 各 SQS API 呼び出しには HTTP コネクションの確立が必要
   - コネクション確立にはオーバーヘッドがある（TCP ハンドシェイク、TLS ハンドシェイク）
   - コネクションプールはコネクションを再利用でき、パフォーマンスを向上させる

### コネクションプール枯渇が発生する理由

- すべてのコネクションが使用中の場合（例：他の API 呼び出しを処理中）
- 新しい receiveMessage 呼び出しはコネクション解放を待つ必要がある
- 待機時間が `connectionAcquisitionTimeout`（デフォルト2秒）を超えると、タイムアウトが発生する

### 新しい Queue のコネクション要件

**通常トラフィック分析：**

| Queue | receiveMessage | deleteMessage | 実行ごとの総コネクション数 |
|-------|----------------|---------------|---------------------------|
| Queue 1 - Transactional | 1回 | 最大10回 | 最大11個 |
| Queue 2 - Non-transactional SMS | 1回 | 最大10回 | 最大11個 |
| Queue 3 - Email metrics | 実際の使用に依存 | 実際の使用に依存 | 実際の使用に依存 |
| **Queue 4 - 新しい EMAIL queue** | **1回** | **10回** | **11個** |

**コネクションプール要件：**
- **新しい queue 追加前**：既存の3つの queue とそれぞれのトラフィックパターン
- **新しい queue 追加後**：新しい queue が毎秒11個のコネクションを追加
- **総並行コネクション数**：通常動作時に50（デフォルトのコネクションプールサイズ）を容易に超過
- **バーストシナリオ**：コネクション要件が大幅に増加する可能性がある

**バーストトラフィック推定：**
- バーストトラフィックは通常トラフィックの10倍になる可能性がある
- 新しい queue だけでも、バースト時に110コネクション/秒（11コネクション × 10倍）を生成する可能性がある
- 他の queue のトラフィックと組み合わせると、総並行コネクション要件は200を超える可能性がある

## 解決策

### カスタム SQS Client Producer の作成

適切なコネクションプールパラメータでカスタム SQS client Producer (`AwsSqsClientProducer`) を作成しました。

### Production 環境設定

```yaml
aws:
  sqs:
    max-concurrency: 200                    # 200個の並行リクエストを処理
    max-pending-connection-acquires: 2000   # 最大2000個のリクエストをキューイング可能
    connection-timeout-seconds: 3           # コネクション確立タイムアウト 3秒
    connection-acquisition-timeout-seconds: 10  # コネクション取得タイムアウト 10秒
    api-call-timeout-seconds: 15            # API 呼び出し総タイムアウト 15秒
    api-call-attempt-timeout-seconds: 15    # API 呼び出し試行タイムアウト 15秒
```

- 注意：`max-pending-connection-acquires` を明示的に指定しないと、ランタイム例外が発生します。計算の結果、デフォルト値の10,000は高すぎると判断し、少し下げました。

### 設定パラメータの説明

| パラメータ | デフォルト値 | 推奨値 | 説明 |
|-----------|-------------|--------|------|
| max-concurrency | 50 | 200 | 最大並行リクエスト数 |
| max-pending-connection-acquires | 10,000 | 2,000 | 最大キューイングリクエスト数 |
| connection-acquisition-timeout-seconds | 2 | 10 | コネクション取得タイムアウト（秒） |
| connection-timeout-seconds | 10 | 3 | コネクション確立タイムアウト（秒） |
| api-call-timeout-seconds | 無制限 | 15 | API 呼び出し総タイムアウト（秒） |
| api-call-attempt-timeout-seconds | 無制限 | 15 | API 呼び出し試行タイムアウト（秒） |

### 設定パラメータの計算ロジック

**maxConcurrency (200)**：
- 目的：最大並行リクエスト数を設定
- HTTP/1.1 の場合：最大コネクション数に相当
- HTTP/2 の場合：コネクション数は接続あたりの最大ストリーム数に依存
- 推奨値：200（実際のトラフィック要件に基づく）

**maxPendingConnectionAcquires (2000)**：
- 目的：maxConcurrency に達したときにキューイング可能な最大リクエスト数
- 計算ロジック：
  - 通常ケース：複数の queue が並行リクエストを生成し、新しい queue の11コネクション/秒を含む
  - バーストケース：200コネクション/秒を容易に超過する可能性がある
  - 処理時間が0.5秒の場合、バースト時に0.5秒以内に100+個の新しいリクエストが発生
  - 200個のコネクションがすべて使用中の場合、最大100+個のリクエストがキューイングする必要がある
  - 安全マージンを加えて：推奨 `2000`
- 推奨値：`2000`（過度なリソース使用なしでバーストを処理可能）

**connectionAcquisitionTimeout (10秒)**：
- 目的：コネクションプールからコネクションを取得するタイムアウト
- 問題：高トラフィックシナリオでは、2秒ではコネクション解放を待つには不十分
- 推奨値：10秒（リクエストにコネクションを待つ時間をより多く提供）

## なぜ QA 環境では問題が発生しないのか？

### QA 環境の設定差異

| 設定項目 | Production | QA |
|---------|------------|-----|
| Poller 頻度 | 1秒 | 3秒 |
| Transactional batch size | 10 | 10 |
| Non-transactional batch size | 10 | 5 |
| 新しい queue のコネクション要件 | 11コネクション/秒 | 低い（より長い poller 間隔のため） |
| リクエストレート | 高い（複数の queue + 新しい queue） | Production の約1/3 |

### 分析

- QA 環境のリクエストレートは Production より大幅に低い（約1/3）
- より長い poller 間隔（3秒）により、毎秒のコネクション要件が減少
- デフォルトのコネクションプール（50個のコネクション）でも、QA トラフィックを処理するには十分
- より長い poller 間隔により、実行間でコネクションを解放する時間がより多く提供される
- 上記の差異により、QA 環境のロードテストではこの問題を検出できず、Production のトラフィックはほぼ24時間連続で、非ピーク時間でもかなりの情報が流入するため、デフォルト設定でデプロイするとすぐにエラーが発生しました

## 重要な学習ポイント

### コネクションプール位置の重要性

| 特性 | 説明 |
|------|------|
| 位置 | **アプリケーション側**、SQS サービス側ではない |
| 管理 | NettyNioAsyncHttpClient によって管理 |
| ストレージ | アプリケーションメモリ内 |
| 目的 | HTTP コネクションの再利用 |

### コネクションプールの制限

コネクションプールには上限があり、それは `maxConcurrency` です：
- コネクションプール制限 = `maxConcurrency`（現在200）
- 制限に達すると、新しいリクエストはキューイングして待機
- キューイング制限 = `maxPendingConnectionAcquires`（現在2000）
- 待機時間制限 = `connectionAcquisitionTimeout`（現在10秒）

### アーキテクチャ図

```
Communication service (ECS Container)
├── AwsSqsClientProducer
│   └── SqsAsyncClient
│       └── NettyNioAsyncHttpClient  ← コネクションプールはここにあります！
│           └── HTTP Connection Pool (maxConcurrency: 200)
│               ├── Connection 1 ──┐
│               ├── Connection 2 ──┤
│               ├── Connection 3 ──┤
│               └── ...            │
│                                  │
└── When calling SQS API           │
    ├── receiveMessage() ──────────┼──> HTTP コネクションが必要
    ├── deleteMessage() ───────────┼──> HTTP コネクションが必要
    └── sendMessage() ─────────────┼──> HTTP コネクションが必要
                                   │
                                   ▼
                           HTTP/HTTPS connection to AWS
                                   │
                                   ▼
                            SQS Service (AWS)
```

## 予防策

1. **Queue 追加時のコネクション要件評価**
   - 各 queue のコネクション要件を計算（receiveMessage + deleteMessage 数）
   - 総並行コネクション要件が既存のコネクションプール容量を超えるかどうかを評価

2. **コネクションプール使用状況の監視**
   - コネクションプール使用率を監視
   - コネクションプール使用率が閾値を超えたときにアラートを設定

3. **事前のコネクションプール設定調整**
   - 高トラフィック queue を追加する前に、コネクションプール容量を調整
   - バーストトラフィック要件を考慮

4. **テスト環境と Production 環境の設定差異**
   - テスト環境と Production 環境の設定差異を理解
   - テスト環境では問題が発生しないが、Production 環境では発生する可能性があり、負荷テストでもシミュレートできない場合がある

## 結論

- このデバッグ経験は、AWS SQS アプリケーションにおいて、コネクションプールが見過ごされがちだが重要なコンポーネントであることを思い出させます。特に queue を追加する際は、各 queue のコネクション要件パターンを慎重に評価し、サービスが正常に動作することを確保する必要があります。

- Production 環境は、慎重な計算の後にカスタマイズされた設定を適用すべきであり、デフォルト値を直接使用すべきではありません。

- **覚えておくこと**：コネクションプールはアプリケーション側にあり、サービス側ではありません。queue を追加する際、特にコネクション集約型の動作パターンを持つ queue の場合、コネクションプール容量が十分かどうかを評価する必要があります！

## 非技術者向けのレストランのたとえ

HTTP コネクションプールをレストランに例えることができます。

### 元の状況

- レストランにはもともと50席しかありませんでした。
- 各 SQS API request は、レストランで食事をしたいお客さんのような存在です。
- お客さんは座席を取得してからでないと食事を始められません。
- 食事が終わると、座席は解放され、次のお客さんが使えます。

### Queue 追加後

4つ目の queue を追加することは、新しい注文受付窓口を増設したようなものです。

そのため：

- 同時に来店するお客さんの数が増えます。
- もともとの50席はすぐに埋まります。
- 後から来たお客さんは待合エリアで列に並ぶしかありません。
- 長時間待っても座席を得られない場合、お客さんは諦めて帰ってしまいます。

この「長く待った末に諦めて帰る」ことは、システムが次のエラーを投げることに相当します：

```text
Acquire operation took longer than the configured maximum time.
```

---
## connectionAcquisitionTimeout と connectionTimeout の違い

この2つの設定は混同しやすいです。

### connectionAcquisitionTimeout

意味：
> request が、connection pool 内の利用可能な connection を1つ取得するために、最大どれだけ待つかを示します。

レストランのたとえ：
> お客さんが入口で、座席が空くまで最大どれだけ待つか。

今回のエラーはこの段階で発生しました。

### connectionTimeout

意味：
> connection リソースを取得した後、実際にリモートサービスと TCP/TLS connection を確立する際に、最大どれだけ待てるかを示します。

レストランのたとえ：
> お客さんがすでに座席を確保した後、スタッフが厨房と連絡を取るのに最大どれだけ時間をかけられるか。

したがって：

```text
connectionAcquisitionTimeout = 10 秒
connectionTimeout = 3 秒
```

この2つを混同してはいけません。

---

## 参考資料
- [Configure the Netty-based HTTP client](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/http-configuration-netty.html) 
- [NettyNioAsyncHttpClient.Builder](https://sdk.amazonaws.com/java/api/latest/software/amazon/awssdk/http/nio/netty/NettyNioAsyncHttpClient.Builder.html)
