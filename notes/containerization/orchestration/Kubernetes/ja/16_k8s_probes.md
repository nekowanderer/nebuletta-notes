# Kubernetesにおける各種Probe

[English](../en/16_k8s_probes.md) | [繁體中文](../zh-tw/16_k8s_probes.md) | [日本語](../ja/16_k8s_probes.md) | [インデックスに戻る](../README.md)


Kubernetesにおいて、ProbeはKubeletがコンテナに対して定期的に実行するヘルスチェックであり、アプリケーションの信頼性を確保するためのものです。異なるチェック目的に基づいて、Probeは主に3つのタイプに分類されます。

### Probeの概要

*   **`StartupProbe`（スタートアップ探査）**: コンテナ内のアプリケーションが「起動完了」しているかを判定します。これは起動の遅いアプリケーションに特に有用で、準備完了前に他のProbeによって誤って終了されることを防ぎます。
*   **`LivenessProbe`（生存探査）**: アプリケーションがまだ「動作中で健全」かを判定します。探査に失敗した場合、Kubeletはコンテナを再起動し、主に「自己監視」に使用されます。
*   **`ReadinessProbe`（準備完了探査）**: アプリケーションが「トラフィックを受信する準備ができている」かを判定します。探査に失敗した場合、Kubernetesはリクエストの送信を一時停止します。主に「外部依存関係の監視」に使用され、データベース接続や他のAPIなどが対象です。

### Probeフロー図

この図は、Pod起動から各種Probeが動作開始するまでの完全なライフサイクルを描写しています：

```text
+-----------------+
| Pod             |
| Initialization  |
+-----------------+
        |
        v
+------------------------------------------------------------------+
| StartupProbe                                                     |
| （アプリケーションが起動したかをチェック）                              |
|                                                                  |
| [成功] ---------------------> LivenessProbe & ReadinessProbe開始  |
| [失敗] ---------------------> [Podを再起動]                        |
+------------------------------------------------------------------+
        |
        | (StartupProbe成功後)
        |
        +----------------------------------------------------------+
        |                                                          |
        v                                                          v
+-----------------------------+                     +------------------------------+
| LivenessProbe               |                     | ReadinessProbe               |
| (アプリ自体がフリーズしてないか) |                     | (外部依存関係が準備完了か)       |
|                             |                     |                              |
| [失敗] -> [Podを再起動]       |                     | [失敗] -> [Unreadyとマーク]    |
|                             |                     |              (トラフィック停止) |
| [成功] -> (継続監視)          |                     |              |               |
|                             |                     |              v               |
|                             |                     | [成功] -> [Readyとマーク]      |
|                             |                     |              (トラフィック再開) |
+-----------------------------+                     +------------------------------+
```

### 各Probeの特性

| 特性 | StartupProbe（スタートアップ探査） | LivenessProbe（生存探査） | ReadinessProbe（準備完了探査） |
| :--- | :--- | :--- | :--- |
| **用途** | 主に長い起動時間が必要なアプリケーション用。起動完了前に`LivenessProbe`によって誤って終了されることを防ぐ。 | アプリケーションが実行状態であることを確保し、デッドロックなどの問題解決によく使用される。 | アプリケーションがトラフィックを受信する「準備ができている」かを判定し、サービス起動時の依存関係チェックによく使用される。 |
| **監視対象** | アプリケーション自体 | アプリケーション自体 | アプリケーションの外部依存関係<br/>（例：DB接続、他のAPI） |
| **失敗時の動作** | KubeletがPodを再起動 | Kubeletがコンテナを再起動 | Endpoint ControllerがPodをServiceから削除し、トラフィックを一時停止。 |
| **成功時の動作** | もう実行されず、後続のProbeに引き継ぐ。 | コンテナは継続実行され、定時監視される。 | PodがServiceに戻され、トラフィックの受信を開始（または再開）する。 |

### 推奨事項と設定

Probeを設計する際は、以下のベストプラクティスに従うことができます：

#### どのProbeをいつ使用するか？

1.  **`StartupProbe`**: アプリケーションの起動時間が非常に長く、`LivenessProbe`の`failureThreshold * periodSeconds`を超える場合に使用すべきです。
2.  **`LivenessProbe`**: ほぼすべての長時間実行サービスで設定すべきです。アプリケーションのクラッシュやデッドロックなどの問題を捕捉できます。ただし、探査ロジックは可能な限り軽量であるべきで、アプリケーションに追加の負荷を与えることを避けるべきです。
3.  **`ReadinessProbe`**: アプリケーションが外部システム（データベース、キャッシュ、他のAPIなど）に依存している場合、または起動後にサービス提供前に初期化作業を実行する必要がある場合、`ReadinessProbe`を使用する必要があります。これにより、完全に準備完了時のみトラフィックが流入し、グレースフルなサービス起動とローリングアップデートが実現されます。

#### Probe Endpoint設計の違い

よくある間違いは、Liveness ProbeとReadiness Probeが同じヘルスチェックエンドポイントを使用することです。これは悪い実践です。なぜなら、それらの関心事が異なるからです：

*   **Liveness Endpoint（`/healthz`、`/livez`）**: アプリケーション自体の状態のみを報告すべきで、外部依存関係をチェックすべきではありません。データベースが一時的に使用不可のためにLiveness Probeが失敗すると、Podが不必要に再起動され、連鎖的な再起動嵐を引き起こす可能性があります。
*   **Readiness Endpoint（`/readyz`、`/ready`）**: アプリケーションが完全なサービスを提供するために必要なすべて、バックエンドデータベースや他のサービスとの接続を含めてチェックすべきです。バックエンドサービスに問題がある時、Readiness Probe失敗によりPodが一時的にオフラインになり、バックエンドサービス回復後に自動的にオンラインに戻ります。これはより優雅な処理方法です。

#### 設定例

これは3種類のProbeすべてを組み合わせたPod設定例です：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: my-app-container
    image: my-app:1.0
    ports:
    - containerPort: 8080
    
    # StartupProbe: アプリケーションに最大5分間（30 * 10秒）の起動時間を与える
    startupProbe:
      httpGet:
        path: /api/health/startup
        port: 8080
      failureThreshold: 30
      periodSeconds: 10

    # LivenessProbe: 起動後、15秒毎に生存確認
    livenessProbe:
      httpGet:
        path: /api/health/liveness
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 15
      timeoutSeconds: 2
      failureThreshold: 3

    # ReadinessProbe: 起動後、20秒毎にサービス準備完了確認
    readinessProbe:
      httpGet:
        path: /api/health/readiness
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 20
      timeoutSeconds: 5
      failureThreshold: 3
```