# EKS Fargate Cluster のアーキテクチャとメカニズム

[English](../en/15_aws_eks_fargate_cluster.md) | [繁體中文](../zh-tw/15_aws_eks_fargate_cluster.md) | [日本語](../ja/15_aws_eks_fargate_cluster.md) | [インデックスに戻る](../README.md)

### EKS Fargate とは

EKS Fargate は Amazon が提供するサーバーレスコンテナ実行方式で、ユーザーが EC2 インスタンスを管理する必要なく Kubernetes Pod を実行できます。Fargate は各 Pod に適切な計算リソースを自動的に割り当て、基盤となるインフラストラクチャ管理を完全に引き受けます。

### アーキテクチャ図
```                                                                                                                                                                                                                                                                        
                                                           AWS EKS                                                                       

              Admin EC2                                    Cluster                                     Fargate Computing                 
   +-------------------------------+    +------------------------------------------+      +------------------------------------------+   
   | +--------+                    |    |                                          |      | +-------------+  +---------------------+ |   
   | | eksctl |                    |    |                                          |      | | Master Node |  | Master Replica Node | |   
   | +--------+                    |    | +-------------+      +-----------------+ |      | +-------------+  +---------------------+ |   
   |  |                            |    | | Pod: my-pod |      | Pod: my-pod     | |      +------------------------------------------+   
   |  |--create k8s cluster -------|--->| | ns: app-ns  |      | ns: kube-system | |                                                     
   |  |                            |    | +------|------+      +--------|--------+ |                   Fargate Computing                 
   |  +--create cluster resources -|--->|        |                      |          |      +------------------------------------------+   
   |                               |    |        |                      |          |      |  +-------------+     +-------------+     |   
   +-------------------------------+    +--------|----------------------|----------+      |  | Worker Node |     | Worker Node |     |   
                                                 \                      \                 |  +-------------+     +-------------+     |   
                                                  |                      |                |     +-------------+     +-------------+  |   
                                         +-----------------+    +-----------------+       |   ->| Worker Node |     | Worker Node |  |   
                                         | Fargate Profile |    | Fargate Profile |-------|--/  +-------------+     +-------------+  |   
                                         +-----------------+    +-----------------+       |                                          |   
                                                  |                                       |    +-------------+    +-------------+    |   
                                                  ----------------------------------------|--->| Worker Node |    | Worker Node |    |   
                                                                                          |    +-------------+    +-------------+    |   
                                                                                          +------------------------------------------+   
```

### コアアーキテクチャコンポーネント

アーキテクチャ図によると、EKS Fargate には以下の主要コンポーネントが含まれています：

- **Resource Provider**：左側の AWS リソースプロバイダー、Admin EC2 と eksctl ツールが含まれる
- **Kubernetes Cluster**：中央の K8s クラスター、各種 Pod と Namespace が含まれる
- **Fargate Profile**：どの Pod が Fargate で実行されるかを定義する Fargate 設定
- **Fargate Computing**：実際の Fargate 計算リソース、Master Node と Worker Node に分かれる

### EKS Fargate の動作メカニズム

```
Admin EC2 → eksctl → Kubernetes Cluster → Fargate Profile → Fargate Computing
```

1. **クラスター作成**：Admin EC2 上の eksctl ツールを使って K8s クラスターを作成
2. **リソース構成**：eksctl がクラスターに必要な各種リソースを作成
3. **Pod デプロイ**：Pod が指定された Namespace（`app-ns`、`kube-system` など）にデプロイされる
4. **Profile マッチング**：Fargate Profile が設定されたルールに基づいて条件に合う Pod を選択
5. **リソース割り当て**：選択された Pod が Fargate Computing リソースに割り当てられて実行

### Fargate Profile の役割

Fargate Profile は、どの Pod が Fargate で実行されるかを決定する重要な設定です：

- **Namespace 選択**：特定の Namespace（図中の `app-ns` など）を指定可能
- **Pod セレクター**：Label Selector を通じてどの Pod が Fargate を使用するかを精密に制御
- **リソース分離**：異なるワークロードが適切な計算環境で実行されることを保証

### Fargate と従来の EC2 の違い

- **ノード管理不要**：Fargate が基盤となる計算リソースを自動管理し、EC2 インスタンスの手動設定が不要
- **従量課金**：実際に実行されている Pod のみに課金され、アイドル状態のノードに対する課金なし
- **自動スケーリング**：Fargate が Pod の要求に基づいて適切な計算リソースを自動割り当て
- **高いセキュリティ**：各 Pod が独立した計算環境で実行され、より良い分離性を提供

### 使用場面と考慮事項

**Fargate の使用に適した場面：**
- EC2 インスタンスの複雑さを管理したくない場合
- 断続的または予測不可能な特性を持つワークロード
- 高速起動・停止が必要なアプリケーション
- セキュリティ分離の要求が高い環境

**注意事項：**
- Fargate のコストは EC2 より高くなる可能性があり、特に長時間実行されるワークロードの場合
- 一部の K8s 機能が Fargate で制限される可能性がある
- Pod の正しいスケジューリングを確保するため、適切な Fargate Profile 設計が必要