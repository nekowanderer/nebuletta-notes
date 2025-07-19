
# 關於 kubelet

[English](../en/37_about_kubelet.md) | [繁體中文](../zh-tw/37_about_kubelet.md) | [日本語](../ja/37_about_kubelet.md) | [回到索引](../README.md)

## kubelet 是什麼？
`kubelet` 是跑在每個 Kubernetes Node 上的 **本地守護程式（agent）**，負責與 K8s API Server 溝通，並**確保 Pod 被正確執行在該 Node 上**。

---

## 核心職責
1. **接收 Pod 定義（PodSpec）**  
   從 API Server 收到要執行的 Pod 描述。
2. **啟動並監控容器**  
   通常透過 container runtime（如 Docker 或 containerd）來執行。
3. **回報狀態**  
   將 Pod 與 Node 的健康狀態、資源使用情況，回傳給 API Server。
4. **健康檢查**  
   執行 liveness/readiness probes，確認容器健康與否。
5. **處理生命週期**  
   包含 Pod 啟動、終止、重啟等行為。

---

## kubelet 不負責的事
- 不負責排程（這是 kube-scheduler 的工作）
- 不管理其他 Node（它只管理自己所在的 Node）

---

## 名稱由來
- `kubelet` = `kube`（Kubernetes）+ `let`（小的）
- `let` 是英文中的 **diminutive（指小詞）** 後綴
  - 常見例子：
    - `piglet`（小豬）
    - `booklet`（小冊子）
    - `leaflet`（傳單）
- 所以 `kubelet` 的意思是：
  > **「Kubernetes 的小成員」或「小守護程式」**

---

## 總結
| 項目 | 說明 |
|------|------|
| 功能 | 管理本機 Node 上的 Pod |
| 通訊對象 | 與 K8s API Server 互動 |
| 運行位置 | 每台 Node 各跑一個 |
| 名稱意義 | Kubernetes 的小單位守護程式 |

---

## 延伸閱讀建議
- kubelet 與 container runtime 的互動機制（Container Runtime Interface, CRI）
- kubelet 與 kube-proxy 的差異
- kubelet 的啟動參數與 systemd 管理方式
