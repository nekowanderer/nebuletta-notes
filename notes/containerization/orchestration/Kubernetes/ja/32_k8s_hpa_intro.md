# Kubernetes HPA（Horizontal Pod Autoscaling）の概要

[English](../en/32_k8s_hpa_intro.md) | [繁體中文](../zh-tw/32_k8s_hpa_intro.md) | [日本語](../ja/32_k8s_hpa_intro.md) | [インデックスに戻る](../README.md)

Kubernetes HPAは、CPU使用率などのリソース使用状況に基づいてPodの数を自動調整し、アプリケーションの柔軟なスケールアップ・スケールダウンを実現します。その動作フローは以下の通りです：

- HPAはDeployment内のすべてのPodのCPU使用率を継続的に監視します。
- これらのリソース使用データはMetric Serverが収集を担当します。Metric Serverは各Podから定期的にCPU（またはメモリなど）の使用情報を収集し、これらのデータをHPAに提供します。
- HPAはこれらのリアルタイム監視データを取得した後、事前定義された閾値（目標CPU使用率など）に基づいてReplicaSet内のPodの数を調整する必要があるかを判断します（例：3個のPodから増減）。
- 負荷が上昇した際、HPAは自動的にPod数を増やし、負荷が下降した際は自動的に減らし、リソース利用率とサービスの安定性を向上させます。

## 重要コンポーネントの説明
- **Cluster**：Kubernetesクラスタ環境
- **Deployment**：アプリケーションのデプロイとアップグレードを管理
- **ReplicaSet**：指定された数のPodの継続的な稼働を保証
- **Pod**：アプリケーションが実行される最小単位、1つまたは複数のContainerを含む
- **Metric Server**：クラスタ内の各Podのリアルタイムリソース使用データを収集・提供し、HPAの参考とする
- **HPA**：Metric Serverが提供するデータに基づいてPod数を自動調整

## Metric ServerとPrometheusの比較

- **Metric Server**：Kubernetesに内蔵された軽量なリソース監視コンポーネント。主にHPAやその他の内部コンポーネントにCPU、メモリなどの基本的な指標をリアルタイムで提供します。インストールが簡単で、基本的な自動スケーリング機能のみが必要なシナリオに適しています。
- **Prometheus**：より強力な監視システムで、より詳細で多様な指標（カスタムアプリケーション指標を含む）を収集でき、長期的なデータ保存、複雑なクエリとアラートをサポートします。kube-prometheus-stackやカスタムAdapterなどをインストールすることで、HPAがPrometheusから直接指標を取得し、より高度な自動スケーリングを実行できます。

**まとめ**：
基本的なHPA機能のみが必要な場合はMetric Serverで十分です。より高度な監視と自動スケーリング要件がある場合は、Metric ServerをPrometheusに置き換え、カスタム指標を使用した柔軟な調整を検討できます。

## Metrics Serverのプルとプッシュモデル

Kubernetesに内蔵されたMetric ServerやPrometheusなどの主流監視システムは、主に「プル（Pull）」モデルを採用しています。つまり、Prometheusは各Podやサービスの`/metrics` HTTPエンドポイントに定期的にリクエストを送信し、リアルタイム監視データを収集します。このアプローチの利点は以下の通りです：
- 監視システムが収集頻度とターゲットを統一管理し、管理と調整が容易
- ターゲットサービスがオフラインになったかを即座に検出可能（データがプルできないため）
- サービスディスカバリ（Kubernetes Service Discoveryなど）と連携して動的に変化するターゲットを自動追跡

一方、他の監視システムやデータベース（ClickHouse、Graphiteなど）では「プッシュ（Push）」モデルをサポートしています。このモードでは、アプリケーションがHTTP、UDPなどのプロトコルを通じて監視データを指定されたmetricsサーバーに能動的に送信します。このアプローチの利点は以下の通りです：
- 短命なタスク（バッチジョブなど）に適しており、タスク終了時に一度データを能動的にプッシュできる
- 複数の受信者にデータを同時にプッシュすることが容易

### 具体例
- **Prometheus**：デフォルトで「プル（Pull）」モデル、ターゲットのmetricsエンドポイントを定期的に能動的に取得。プッシュサポートが必要な場合は、Pushgatewayを仲介として使用可能
- **ClickHouse**：Prometheusのremote-write（Push）プロトコルをサポートし、アプリケーションがHTTP経由でmetricsを直接ClickHouseにプッシュ可能
- **Graphite**：「プッシュ（Push）」モデルをネイティブサポート、アプリケーションが直接Graphiteにデータを送信

**まとめ**：
- **プル（Pull）モデル**：Prometheus、Kubernetes Metric Server
- **プッシュ（Push）モデル**：ClickHouse（remote-write）、Graphite
- 実務では両モデルにそれぞれ利点と欠点があり、アプリケーションシナリオとインフラ要件に基づいて柔軟に選択可能です。