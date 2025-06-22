# K8S Service Types

[English](../en/11_k8s_service_types.md) | [繁體中文](../zh-tw/11_k8s_service_types.md) | [日本語](../ja/11_k8s_service_types.md) | [回到索引](../README.md)

### ClusterIP
| 重點   | 說明                                                                            |
| ---- | ----------------------------------------------------------------------------- |
| 角色   | 叢集中 Pod 之間的「室內分機」                                                             |
| 入口   | 只有叢集內部可達（kube-proxy + iptables / IPVS DNAT）                                   |
| DNS  | kubernetes 內建的 CoreDNS 會自動配 A/AAAA 記錄，Pod 用 `<svc>.<ns>.svc.cluster.local` 互叫 |
| 適合情境 | 微服務互相呼叫、內部 API、資料庫、Metrics、Queue 等完全不打算直接暴露到外部的服務                             |

- 優點
  - 預設就有，設定最少。
  - Pod 滾動更新時 IP 變了也不用改下游程式碼，透過 Service 名稱自動轉。
- 缺點
  - 叢集外完全連不到；測試或除錯要額外做 port-forward 或開臨時 Service。


### NodePort
| 重點   | 說明                                                     |
| ---- | ------------------------------------------------------ |
| 角色   | 把每個 Worker Node 當作臨時「大門房」：在 30000-32767 的範圍開固定埠口       |
| 入口   | 外部使用者 → `任一節點 IP:NodePort` → 叢集 Service → Pod          |
| 適合情境 | PoC、內網 lab、沒有雲端 Load Balancer、但又急著讓外部（或 VPN 使用者）進來看的服務 |

- 優點
  - 不需要雲端整合，純 K8s 功能就能用。
  - 一行 YAML 就能把服務推到每台節點對外。
- 缺點
  - Port 號不好看也不好記；公司防火牆還得放行這些高號段。
  - 若節點數很多，每台都聽同一埠，資安審計變複雜。
  - 沒有健康檢查、SSL、L7 轉發等高級功能。


### LoadBalancer
| 重點   | 說明                                                                 |
| ---- | ------------------------------------------------------------------ |
| 角色   | 叫雲端供應商（AWS ELB、GCP TCP/HTTP LB、Azure LB…）自動生成一個 L4/7 Load Balancer |
| 入口   | 外部使用者 → LB 公網 IP / DNS → 轉到 NodePort（由雲端自動管理）→ Pod                 |
| 適合情境 | 生產環境、需要固定公網 DNS、TLS 終端、健康檢查、量大、跨 AZ 的高可用流量                         |

- 優點
  - 一個 Service 一個雲端 LB，外部直接用 DNS 名稱即可。
  - 原生支援健康檢查、SSL offload、Cross-Zone / AZ HA。
  - 可以搭配 ExternalTrafficPolicy=Local 做 client IP preservation。
- 缺點
  - 每個 LB 都是錢；大量微服務會很貴。
  - 侷限在支援 LB 的雲環境；裸機或 on-prem 需要 MetalLB、BGP LB 或自架。
  - 建立/摧毀 LB 需要等待雲端 API，部署時間比 ClusterIP & NodePort 長。


### 比較
| 功能面向     | ClusterIP | NodePort    | LoadBalancer    |
| -------- | --------- | ----------- | --------------- |
| 外部可達性    | 否         | 是（IP\:Port） | 是（公網/私網 LB DNS） |
| 連線易記度    | ★★★       | ★☆（高埠號）     | ★★★             |
| 成本       | 無         | 無           | 每顆 LB 需額外付費     |
| TLS／健康檢查 | Pod 內自行處理 | Pod 內自行處理   | LB 可代勞          |
| 雲端依賴     | 無         | 無           | 高（需雲供應商整合）      |
| 常見用途     | 內部微服務     | 測試、Lab      | 產品線對外 API、Web   |

- LoadBalancer 是雲端對外門面
- NodePort 是它通往叢集的統一後門
- Service 的真正轉送工作都靠節點上的 kube-proxy/iptables 做完。


### LoadBalancer 為什麼會轉到 NodePort? 
| 行為                                                                | 發生在哪裡                     | 為什麼要這樣做？                                              |
| ----------------------------------------------------------------- | ------------------------- | ----------------------------------------------------- |
| 建立 `type: LoadBalancer` 的 Service                                | Kubernetes API Server ✅  | 只是宣告「我要一個雲端 LB」                                       |
| Cloud Controller Manager (CCM) 呼叫雲供應商 API                       | 控制平面 ✅                   | 生一個 ELB / NLB / L4 LB…                                |
| **同時** kube-proxy 對所有 Worker Node 開 **NodePort**（30000-32767 取一號） | **每台 Worker Node** (資料平面) | 給 LB 有統一的落地點 <br>→ LB 只要知道「節點 IP : NodePort」即可把流量打進叢集 |

- 因此： LoadBalancer ➜ NodePort ➜ ClusterIP ➜ Pod。NodePort 是雲端 LB 與叢集之間的「共通語言」。


### 所謂的 service，是不是基本上都運作在 worker node 裡面? 
- Service 不是一個 Pod 或程序，其本身沒有 container；它是一組 iptables / IPVS 規則，由 kube-proxy 佈到每台 Worker Node。
- kube-proxy（或 eBPF/XDP 方案）會在 每個 worker node 動態下 iptables/IPVS/eBPF 規則，把：
  - 節點收進來的 `node-ip:nodeport` 或容器內呼叫的 `clusterip:port` 轉寫（DNAT）到真正的 Pod IP / 埠，再配合 round-robin、權重或 session-affinity。
- 也就是說，不論你打的是 ClusterIP、NodePort、還是外部 LB 打進來，封包最終都在節點上被 DNAT 成 Pod IP。
- 因此「Service 運作在 worker node」可理解為：它靠節點層的網路規則在每個節點常駐，而不是某台特定主機上一個伺服器程序。


### NodePort 預設是指 worker node 開放讓外部或是 LB 可以連到的地方嗎?
| 特性   | 說明                                                                                        |
| ---- | ----------------------------------------------------------------------------------------- |
| 位置   | 每台 Worker Node 都會 `LISTEN 0.0.0.0:<nodePort>`（kube-proxy 負責）                              |
| 預設範圍 | `30000-32767`（可透過 `--service-node-port-range` 改）                                          |
| 暴露來源 | - 直接宣告 `type: NodePort` 讓外部或 VPN 使用者打 <br> - 被 `type: LoadBalancer` 隱含建立，僅讓雲端 LB 打 |
| 流量動線 | `節點 IP:NodePort` → 送入 `service` 的 DNAT → 隨機選一個後端 Pod（RR / IPVS LC …）                      |
| 常見誤會 | **NodePort ≠ 開在 Pod**；它開在節點網卡上。Pod 只收最終 DNAT 後的封包                                         |
| 外部一定打得到嗎？ | **還要看防火牆 / Security Group / NACL** 是否放行；K8s 只綁埠，不幫你開安全規則                             |

### 簡化的關係圖
```
External client
     |
     |  (LoadBalancer DNS / IP)
[Cloud LB]  <----------------- 只有 type=LoadBalancer 會出現
     |
nodeIP:NodePort  <------------ NodePort 與 LoadBalancer 共用
     |
ClusterIP:Port   <------------ 叢內流量（Service 名稱解析）
     |
PodIP:Port       <------------ 真正容器

```

### References
- [Kubernetes Official Document - Service Type](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)