# DNAT and SNAT in K8S

[English](../en/12_dnat_and_snat_in_k8s.md) | [繁體中文](../zh-tw/12_dnat_and_snat_in_k8s.md) | [日本語](../ja/12_dnat_and_snat_in_k8s.md) | [インデックスに戻る](../README.md)

### DNAT
Destination Network Address Translation（宛先ネットワークアドレス変換）
- パケットの宛先IP/Portを別のアドレスに書き換える。
- 従来のファイアウォール/NATルーター：外部の203.0.113.10:80を内部の192.168.1.100:80に変換することが多い。
- Kubernetes：nodeIP:NodePortやClusterIP:portに到達するトラフィックを実際のPodIP:containerPortに変換する。

```
  External Client
          |
          v
203.0.113.10:32001  (NodePort)
          |
          v
    [DNAT 生成]
          |
          v
10.244.3.42:8080  (Pod)

```

##### なぜDNATが必要か？
- 抽象化：Service / NodePort / LoadBalancerが「消えない住所」を提供；実際のPod IPが再構築時に変更されても問題ない。
- 負荷分散：同じエントリアドレスで複数のバックエンドPodにラウンドロビンできる。

##### K8sでのDNATの実装方法
- kube-proxy (iptables/IPVS)
  - natテーブルのPREROUTING（入り口）またはOUTPUT（ローカル生成パケット）チェーンに-j KUBE-SVC-XXXXを注入。
  - KUBE-SVC-XXXXが-j KUBE-SEP-YYYY（各SEPはPodを表す）を1つ選択。
  - KUBE-SEP-YYYYが最後に-j DNAT --to-destination PodIP:Portを実行。
- eBPF/XDP (Cilium, Calico eBPF)
  - カーネルBPFプログラムを使用してより早いフック（TC / XDP）で書き換え、パフォーマンスが向上。

### SNAT
Source Network Address Translation（送信元ネットワークアドレス変換）
- パケットの送信元IP/Portを別のアドレスに書き換える ── DNATが「宛先」を変更するのと対照的
- 戻りパケットが「道を覚えて」戻れるようにする、または複数の内部アドレスを1つの外部アドレスに統合する
- 一般的な用語：SNAT、MASQUERADE（動的SNAT）、NAPT（IPとPortを同時に変更）

##### Kubernetesでの実用的なシナリオ
- Podがインターネットにアクセス（Pod → Internet）
  - Podアドレスは通常10.x/172.x/192.168.xのプライベートネットワークセグメントで、パブリックインターネットルーティングでは認識されない。
  - Worker Nodeがnat POSTROUTINGチェーンでSNAT --to-source nodeIPまたはMASQUERADEを実行し、送信元をノードのパブリックIPに変更。
  - 外部の戻りパケットはノードIPのみを見て、ノードに送り返され、conntrackで対応するPodを見つける。
- Service type=NodePort / LoadBalancerの戻りフロー
  - ExternalTrafficPolicy=Cluster（デフォルト）の場合、ノードが外部トラフィックをPodにDNATした後、送信元もSNAT → PodはノードIPのみを見る。
  - 利点：Podの戻りパケットはノードに戻るだけでよい；欠点：元のClient IPが失われる。
- ノード間Pod to Pod（一部のCNIプラグイン）
  - CNIがトンネルやオーバーレイを使用する場合、ノード出口で再度SNATし、安定した戻りパスを確保。

##### iptablesルールはどのようなものか？
```bash
# kube-proxyが生成する例
-A POSTROUTING -s 10.244.0.0/16 ! -d 10.244.0.0/16 \
        -j MASQUERADE        # Podの送信トラフィックの送信元をNode IPに変更
```
- POSTROUTING：パケットがローカルマシンを離れる直前の最後の瞬間。
- MASQUERADE：ノードの現在の外部IPを動的に選択（IPが変更される可能性のあるクラウドノードに適している）。
- ip-masq-agentやCNIの--masq-outbound=falseでこの動作を無効化できる。

### DNAT v.s. SNAT
| 名前                    | 目的               | 使用タイミング               | 一般的な実行場所 |
| --------------------- | ---------------- | ------------------- | ------------------- |
| **DNAT**              | 宛先IP/Portを変更 | トラフィックをバックエンドPodに転送         | `PREROUTING` / `OUTPUT` |
| **SNAT / MASQUERADE** | 送信元IP/Portを変更 | Podが外部に出る時、戻りパケットがノードに到達できるようにする | `POSTROUTING`           |

### よくある質問
- 実際のClient IPを保持できるか？
  - ServiceをExternalTrafficPolicy=Localに設定してSNATを回避；ただし「ローカルに対応するPodがある」ノードにトラフィックが着地した場合のみ有効。
  - Ingress Controller（Nginx、Traefik）は通常X-Forwarded-Forを読み取って元のIPを保持する。
- SNATはボトルネックになるか？
  - 多数の接続では、conntrackテーブルに十分なメモリが必要；eBPF/XDPソリューションでiptablesジャンプコストを削減できる。
  - クラウドNAT Gateway（AWS GWLB、GCP Cloud NAT）でNode SNAT負荷をオフロードできる。
- オンプレミス/ベアメタルクラスターはどうするか？
  - MetalLB + 追加ファイアウォールを使用；またはBGP CNI（Cilium BGP、Calico BGP）でPod CIDRをアップストリームルーティングに直接広告し、SNATを排除。

### NAT/DNAT/SNATの関係

| 名前           | 変更されるフィールド                 | 一般的な方向            | K8sでの代表的なシナリオ              |
| ------------ | -------------------- | --------------- | ------------------------- |
| **NAT** (総称) | 送信元または宛先のIP/Port      | ——              | 包括的な用語             |
| **DNAT**     | **宛先** | "外部→内部"<br>エントリ負荷分散 | `nodeIP:NodePort → PodIP` |
| **SNAT**     | **送信元**      | "内部→外部"<br>出口インターネットアクセス   | `PodIP → nodeIP`          |

- NAT (Network Address Translation)
  - 総称：パケットヘッダーのIP/ポートを変更する動作はすべてNATと呼ばれる。
  - サブカテゴリ：「どのフィールドを変更するか」に基づいてDNATとSNATにさらに分類。
- DNAT (Destination NAT)
  - 要点：宛先を変更し、パケットが「新しいターゲット」に到達できるようにする。
  - 典型的な用途
    - 家庭：ホームルーター:80をNAS:80に転送。
    - K8s：ClusterIP / NodePortをPodIPに転送。
  - iptablesはnat PREROUTING（パケットが入ってきたらすぐに変更）によく現れる。
- SNAT (Source NAT) / MASQUERADE
  - 要点：送信元を変更し、一貫した戻りパスを確保、または多対一（Many → One）のアドレス節約を行う。
  - 典型的な用途
    - ホームNAT：複数のコンピューターが1つのWAN IPを共有してインターネットアクセス。
    - K8s：Podが外部に出る時、ノードIPに変更；外部の戻りパケットが道を見つけられる。
    - ExternalTrafficPolicy=ClusterのNodePort/LoadBalancerはクライアントIPをノードIPで覆う。
  - iptablesはnat POSTROUTING（パケットがローカルマシンを離れる前の最後の変更）によく現れる。

##### Kubernetesに適用される3つの「NATパイプライン」

```
(1) 外部 → LoadBalancer (パブリック/プライベートIP)
                   |  DNAT (L4/L7 LB)
(2) → nodeIP:NodePort                ⟵ 外部クライアントはNodeのみを見る
                   |  DNAT (kube-proxy / eBPF)
(3) → PodIP:containerPort            ⟵ PodはNodeのみを見る
                   |  SNAT (必要に応じて) ── クライアントIPを覆う必要がある場合
(4) ← Pod→Node  (SNATが有効な場合)         ⟵ 同じノード経由の戻りパス

```
- ステップ1–2：クラウドLBが最初のDNATを実行し、トラフィックをノードのNodePortに転送。
- ステップ2–3：kube-proxy/eBPFが2番目のDNATを実行し、トラフィックを実際のPodに転送。
- ステップ3–4：パケットが戻れるようにするため、必要に応じてノードが送信元をノードIPにSNAT（クライアントIPを保持する場合はSNATを無効化し、ExternalTrafficPolicy=Localを設定し、Podがあるノードにのみトラフィックが到達するようにする）。

##### よくある質問
- NAT = DNAT + SNATか？
  - ほぼそうだが、PAT/NAPT（IPとポートを同時に変更）、Hairpin NAT、Double NATなど、本質的にNATの特殊なケースもある。
- 実際のClient IPを保持するには？
  - K8s ServiceのExternalTrafficPolicy=Localを設定してノードSNATを防ぐ；欠点は「ローカルにPodがある」ノードにのみ転送できる。
  - またはIngress/LB層でPROXY Protocol、X-Forwarded-Forを追加してアプリケーション解析。
- DNAT+SNATはパフォーマンスに影響するか？
  - 多数の接続でのiptablesはconntrackテーブルとルールマッチングコストが顕著；eBPFに切り替えるかクラウドNAT GatewayにオフロードすることでCPUとレイテンシーを大幅に削減できる。 