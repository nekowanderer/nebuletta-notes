# K8S Service Types

[English](../en/11_k8s_service_types.md) | [繁體中文](../zh-tw/11_k8s_service_types.md) | [日本語](../ja/11_k8s_service_types.md) | [インデックスに戻る](../README.md)

### ClusterIP
| 重点 | 説明 |
| ---- | ---- |
| 役割 | クラスター内のPod間の「内線」 |
| エントリーポイント | クラスター内部からのみアクセス可能（kube-proxy + iptables / IPVS DNAT） |
| DNS | Kubernetesの組み込みCoreDNSが自動的にA/AAAAレコードを設定し、Podは`<svc>.<ns>.svc.cluster.local`を使用して相互に呼び出し |
| 適切なシナリオ | マイクロサービス間の相互呼び出し、内部API、データベース、メトリクス、キュー、外部に直接公開する予定のないその他のサービス |

- 利点
  - デフォルトで利用可能、最小限の設定で済む。
  - Podのローリングアップデート時にIPが変更されても、下流のコードを変更する必要がない；Service名による自動転送。
- 欠点
  - クラスター外からは完全にアクセス不可；テストやデバッグには追加のport-forwardや一時的なService作成が必要。

### NodePort
| 重点 | 説明 |
| ---- | ---- |
| 役割 | 各Worker Nodeを一時的な「玄関」として使用：30000-32767の範囲で固定ポートを開く |
| エントリーポイント | 外部ユーザー → `任意のノードIP:NodePort` → クラスターService → Pod |
| 適切なシナリオ | PoC、内部lab、クラウドLoad Balancerがないが、外部（またはVPNユーザー）にサービスを素早く見せたい場合 |

- 利点
  - クラウド統合が不要、純粋なK8s機能で動作。
  - 1行のYAMLでサービスを各ノードに外部公開できる。
- 欠点
  - ポート番号が分かりにくく覚えにくい；企業ファイアウォールでこれらの高番号ポートを許可する必要がある。
  - ノード数が多い場合、各ノードが同じポートでリッスンするため、セキュリティ監査が複雑になる。
  - ヘルスチェック、SSL、L7転送などの高度な機能がない。

### LoadBalancer
| 重点 | 説明 |
| ---- | ---- |
| 役割 | クラウドプロバイダー（AWS ELB、GCP TCP/HTTP LB、Azure LB...）を呼び出してL4/7 Load Balancerを自動生成 |
| エントリーポイント | 外部ユーザー → LBパブリックIP / DNS → NodePortに転送（クラウドが管理）→ Pod |
| 適切なシナリオ | 本番環境、固定パブリックDNSが必要、TLS終端、ヘルスチェック、大容量、クロスAZ高可用性トラフィック |

- 利点
  - 1つのService、1つのクラウドLB、DNS名による外部アクセス。
  - ヘルスチェック、SSLオフロード、クロスゾーン/AZ HAのネイティブサポート。
  - ExternalTrafficPolicy=Localと組み合わせてクライアントIP保持が可能。
- 欠点
  - 各LBにコストがかかる；多数のマイクロサービスは高額になる。
  - LBをサポートするクラウド環境に限定；ベアメタルやオンプレミスではMetalLB、BGP LB、または自前のソリューションが必要。
  - LBの作成/破棄にはクラウドAPIの待機が必要、ClusterIP & NodePortよりデプロイ時間が長い。

### 比較
| 機能面 | ClusterIP | NodePort | LoadBalancer |
| ------ | --------- | -------- | ------------ |
| 外部到達性 | 否 | 是（IP:Port） | 是（パブリック/プライベートLB DNS） |
| 接続の覚えやすさ | ★★★ | ★☆（高番号ポート） | ★★★ |
| コスト | 無料 | 無料 | LBごとに追加料金 |
| TLS／ヘルスチェック | Pod内で処理 | Pod内で処理 | LBが代行可能 |
| クラウド依存性 | なし | なし | 高（クラウドプロバイダー統合が必要） |
| 一般的な用途 | 内部マイクロサービス | テスト、Lab | 本番外部API、Web |

- LoadBalancerはクラウドの外部ファサード
- NodePortはクラスターへの統一された裏口
- Serviceの実際の転送作業はすべてノード上のkube-proxy/iptablesで行われる。

### LoadBalancerがNodePortに転送する理由
| 動作 | 発生場所 | なぜこのようにするのか？ |
| ---- | -------- | ----------------------- |
| `type: LoadBalancer`のService作成 | Kubernetes API Server ✅ | 「クラウドLBが欲しい」と宣言するだけ |
| Cloud Controller Manager (CCM)がクラウドプロバイダーAPIを呼び出し | コントロールプレーン ✅ | ELB / NLB / L4 LB...を作成 |
| **同時に** kube-proxyが全Worker Nodeで**NodePort**を開く（30000-32767から1つ選択） | **各Worker Node**（データプレーン） | LBに統一された着地点を提供 <br>→ LBは「ノードIP : NodePort」を知っていればクラスターにトラフィックを送信可能 |

- したがって：LoadBalancer ➜ NodePort ➜ ClusterIP ➜ Pod。NodePortはクラウドLBとクラスター間の「共通言語」。

### Serviceは基本的にworker node内で動作しているのか？
- ServiceはPodやプロセスではなく、コンテナ自体を持たない；kube-proxyが各Worker Nodeにデプロイするiptables / IPVSルールのセット。
- kube-proxy（またはeBPF/XDPソリューション）が各worker nodeにiptables/IPVS/eBPFルールを動的にデプロイし、以下を変換：
  - 入ってくる`node-ip:nodeport`やコンテナ内の`clusterip:port`呼び出しを実際のPod IP / ポートに（DNAT）し、round-robin、重み、またはsession-affinityと組み合わせる。
- つまり、ClusterIP、NodePort、外部LBのどれを叩いても、パケットは最終的にノード上でPod IPにDNATされる。
- したがって「Serviceがworker nodeで動作する」は、特定のホスト上のサーバープロセスではなく、各ノードに常駐するノードレベルのネットワークルールに依存していると理解できる。

### NodePortはデフォルトでworker nodeが外部やLBからの接続を許可する場所か？
| 特性 | 説明 |
| ---- | ---- |
| 場所 | 各Worker Nodeが`LISTEN 0.0.0.0:<nodePort>`する（kube-proxyが担当） |
| デフォルト範囲 | `30000-32767`（`--service-node-port-range`で変更可能） |
| 露出元 | - `type: NodePort`を直接宣言して外部やVPNユーザーがアクセス <br> - `type: LoadBalancer`によって暗黙的に作成され、クラウドLBのみがアクセス |
| トラフィックフロー | `ノードIP:NodePort` → ServiceのDNATに入る → バックエンドPodをランダム選択（RR / IPVS LC...） |
| よくある誤解 | **NodePort ≠ Podで開く**；ノードネットワークインターフェースで開く。Podは最終DNAT後のパケットのみを受信 |
| 外部から必ずアクセス可能か？ | **ファイアウォール / Security Group / NACL**の許可も必要；K8sはポートをバインドするだけで、セキュリティルールの開放は手伝わない |

### 簡略化された関係図
```
External client
     |
     |  (LoadBalancer DNS / IP)
[Cloud LB]  <----------------- type=LoadBalancerの場合のみ出現
     |
nodeIP:NodePort  <------------ NodePortとLoadBalancerで共有
     |
ClusterIP:Port   <------------ クラスター内トラフィック（Service名解決）
     |
PodIP:Port       <------------ 実際のコンテナ

```

### References
- [Kubernetes Official Document - Service Type](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types) 