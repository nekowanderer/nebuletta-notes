# DNAT and SNAT in K8S

[English](../en/12_dnat_and_snat_in_k8s.md) | [繁體中文](../zh-tw/12_dnat_and_snat_in_k8s.md) | [日本語](../ja/12_dnat_and_snat_in_k8s.md) | [回到索引](../README.md)

### DNAT
Destination Network Address Translation（目的端位址轉換）
- 把封包的目的 IP／Port 改寫成另一組位址。
- 傳統防火牆／NAT 路由器：常把外網的 203.0.113.10:80 轉成內網 192.168.1.100:80。
- Kubernetes：把打到 nodeIP:NodePort 或 ClusterIP:port 的流量，改寫成真正 PodIP:containerPort。

```
  External Client
          |
          v
203.0.113.10:32001  (NodePort)
          |
          v
    [DNAT 產生]
          |
          v
10.244.3.42:8080  (Pod)

```

##### 為什麼要 DNAT？
- 抽象化：Service / NodePort / LoadBalancer 提供「不會消失的門牌」；真實 Pod 重建時 IP 變了也 OK。
- 負載均衡：同一個入口位址，可 round-robin 多個後端 Pod。

##### DNAT 在 K8s 裡怎麼做？
- kube-proxy (iptables/IPVS)
  - 在 nat table 的 PREROUTING（入口）或 OUTPUT（本機產生封包）鏈，注入 -j KUBE-SVC-XXXX。
  - KUBE-SVC-XXXX 裡面挑一條 -j KUBE-SEP-YYYY（每個 SEP 代表一個 Pod）。
  - KUBE-SEP-YYYY 最後做 -j DNAT --to-destination PodIP:Port。
- eBPF/XDP (Cilium, Calico eBPF)
  - 用 kernel BPF program 在更早的 hook（TC / XDP）改寫，效能較佳。

### SNAT
Source Network Address Translation（來源端位址轉換）
- 將封包的來源 IP/Port 改寫成另一組位址 ── 與 DNAT 改「目的端」相對
- 讓回程封包能夠「認得路」回來，或把多個內部位址彙整成一個對外位址
- 常見代名詞：SNAT、MASQUERADE（動態 SNAT）、NAPT（同時改 IP 與 Port）

##### 在 Kubernetes 裡的實務場景
- Pod 要上網（Pod → Internet）
  - Pod 位址通常是 10.x/172.x/192.168.x 私有網段，公網路由不認得。
  - Worker Node 在 nat POSTROUTING 鏈做 SNAT --to-source nodeIP 或 MASQUERADE，把來源改成節點的公網 IP。
  - 外網回封包只看得到節點 IP，傳回節點後再由 conntrack 找到對應 Pod。
- Service type=NodePort / LoadBalancer 回流
  - 如果 ExternalTrafficPolicy=Cluster（預設），節點把外部流量 DNAT 到 Pod 之後還會 SNAT 來源 → Pod 只看到節點 IP。
  - 好處：Pod 回包時只要打回節點即可；壞處：失去原始 Client IP。
- 跨 Node Pod 到 Pod（某些 CNI Plugin）
  - 若 CNI 使用隧道或 overlay，可能在節點出口再 SNAT，一樣確保回包路徑穩定。

##### iptables 規則長什麼樣？
```bash
# kube-proxy 產生的範例
-A POSTROUTING -s 10.244.0.0/16 ! -d 10.244.0.0/16 \
        -j MASQUERADE        # 把 Pod 出去的流量來源改成 Node IP
```
- POSTROUTING：封包即將離開本機前最後一刻。
- MASQUERADE：動態選用節點當下的外網 IP（適合雲端節點可能換 IP 的情況）。
- 若有 ip-masq-agent 或 CNI 的 --masq-outbound=false，可以關掉這個行為。

### DNAT v.s. SNAT
| 名稱                    | 目的               | 什麼時機用               | 常在何處執行 |
| --------------------- | ---------------- | ------------------- | ------------------- |
| **DNAT**              | 改目的IP/Port | 把流量導向後端 Pod         | `PREROUTING` / `OUTPUT` |
| **SNAT / MASQUERADE** | 改來源IP/Port | Pod 對外時，確保回包能順利回到節點 | `POSTROUTING`           |

### 常見疑問
- 能保留真實 Client IP 嗎？
  - 把 Service 設為 ExternalTrafficPolicy=Local，就不 SNAT；但僅當流量落到「本機有對應 Pod」的節點才成立。
  - Ingress Controller（Nginx、Traefik）通常會讀 X-Forwarded-For 來保留原 IP。
- SNAT 會不會成為瓶頸？
  - 大量連線時，conntrack table 需要足夠記憶體；eBPF/XDP 方案可減少 iptables 跳轉成本。
  - 雲端 NAT Gateway（AWS GWLB、GCP Cloud NAT）可卸載 Node SNAT 負擔。
- On-prem／裸機叢集怎麼辦？
  - 可以用 MetalLB + 額外防火牆；或 BGP CNI（Cilium BGP、Calico BGP），直接把 Pod CIDR 公告給上游路由，省掉 SNAT。

### NAT/DNAT/SNAT 的關係

| 名稱           | 改寫欄位                 | 常見方向            | 在 K8s 裡的代表場景              |
| ------------ | -------------------- | --------------- | ------------------------- |
| **NAT** (總稱) | 任何來源或目的 IP/Port      | ——              | Umbrella term             |
| **DNAT**     | **目的** (Destination) | 「外→內」<br>入口負載均衡 | `nodeIP:NodePort → PodIP` |
| **SNAT**     | **來源** (Source)      | 「內→外」<br>出口上網   | `PodIP → nodeIP`          |

- NAT (Network Address Translation)
  - 總稱：凡是改掉封包頭裡 IP/埠 的動作都叫 NAT。
  - 子類：根據「改哪個欄位」再分 DNAT 與 SNAT。
- DNAT (Destination NAT)
  - 重點：改目的端，讓封包抵達「新的目標」。
  - 典型用途
    - 在家把 家用路由器:80 指到 NAS:80。
    - K8s 把 ClusterIP / NodePort 指到 PodIP。
  - iptables 常出現在 nat PREROUTING (封包剛進來就先改)。
- SNAT (Source NAT) / MASQUERADE
  - 重點：改來源端，確保回程路徑一致，或做多對一（Many → One）的位址節省。
  - 典型用途
    - 家用 NAT：多台電腦共用一條 WAN IP 出網。
    - K8s：Pod 往外網走時，改成節點 IP；外網回封包才找得到路。
    - NodePort／LoadBalancer 若採 ExternalTrafficPolicy=Cluster 時，把 client IP 一併覆蓋成 node IP。
  - iptables 常出現在 nat POSTROUTING (封包要送出本機前最後再改)。

##### 套用到 Kubernetes 的三條「NAT 管線」

```
(1) 外部 → LoadBalancer (公網/私網 IP)
                   |  DNAT (L4/L7 LB)
(2) → nodeIP:NodePort                ⟵ 外部客戶端只看到 Node
                   |  DNAT (kube-proxy / eBPF)
(3) → PodIP:containerPort            ⟵ Pod 只看到 Node
                   |  SNAT (視需求) ── 若要把 client IP 蓋掉
(4) ← Pod→Node  (SNAT 若開)         ⟵ 回程路徑同節點

```
- 步驟 1–2：雲端 LB 做第一次 DNAT，把流量導向節點的 NodePort。
- 步驟 2–3：kube-proxy/eBPF 再做一次 DNAT，把流量導向真正的 Pod。
- 步驟 3–4：為了讓封包回得去，節點視情況 SNAT 將來源改成 node IP（若保持 client IP 就關掉 SNAT，設定 ExternalTrafficPolicy=Local 並讓流量只打有 Pod 的節點）。

##### 常見疑問
- NAT = DNAT + SNAT 嗎？
  - 幾乎是，但還有 PAT/NAPT（同時改 IP 和埠）、Hairpin NAT、Double NAT 等衍生用語，本質上仍是 NAT 的特例。
- 如何保留真實 Client IP？
  - K8s Service 設 ExternalTrafficPolicy=Local，讓節點不 SNAT；缺點是只能轉發到「本機有 Pod」的節點。
  - 或在 Ingress / LB 層加 PROXY Protocol、X-Forwarded-For，讓應用程式解析。
- DNAT+SNAT 會不會影響效能？
  - iptables 在大量連線下，conntrack Table 與 Rule 匹配成本顯著；改用 eBPF 或對外卸載到 雲端 NAT Gateway 可大幅降低 CPU 與延遲。
