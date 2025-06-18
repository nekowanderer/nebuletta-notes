# 關於 Taint 

[English](../en/04_about_taint.md) | [繁體中文](../zh-tw/04_about_taint.md) | [日本語](../ja/04_about_taint.md) | [回到索引](../README.md)

---

## 1. Master Node 可以有 Pod 嗎？

- **理論上可以，但預設情況下不建議也不會這麼做。**
- Master Node 主要負責叢集管理（API Server、Scheduler、Controller Manager 等），不是用來跑一般應用程式 Pod。
- 預設會加上一個 taint（如 `node-role.kubernetes.io/master:NoSchedule` 或 `node-role.kubernetes.io/control-plane:NoSchedule`），阻止 Scheduler 將一般 Pod 排程到 Master Node。
- 如果手動移除 taint，Master Node 也可以執行 Pod，這在測試環境（如 minikube、k3s）很常見。
- 生產環境建議 Master Node 只跑控制元件，Worker Node 才負責跑應用程式的 Pod。

---

## 2. Taint 的概念是什麼？

- **Taint（污點）** 是加在 Node 上的限制條件，用來控制哪些 Pod 可以被排程到這個 Node。
- **Toleration（容忍）** 則是加在 Pod 上，表示這個 Pod 可以容忍某種 taint，可以被排程到有這個 taint 的 Node。

### Taint 格式
```
key=value:effect
```
- **key**：自訂名稱
- **value**：自訂值
- **effect**：影響方式
  - `NoSchedule`：沒有對應 toleration 的 Pod 不會被排程到這個 Node
  - `PreferNoSchedule`：盡量不要排程，但不是絕對禁止
  - `NoExecute`：不僅不排程，還會把已經在這個 Node 上、沒有 toleration 的 Pod 驅逐（Evict）掉

### 實際例子
- Master Node 預設 taint：
  ```
  node-role.kubernetes.io/master:NoSchedule
  ```
  沒有加上對應 toleration 的 Pod，不能被排程到這個 Node 上。

### 什麼時候會用到 taint？
- 保護 Master Node，不讓一般應用程式的 Pod 跑上去
- 有特殊硬體（如 GPU）的 Node，只讓特定 Pod 跑
- 做資源隔離、維護、測試等

---

### 關於 `node-role.kubernetes.io/master:NoSchedule` 的格式

- 雖然標準格式是 `key=value:effect`，但 **value 可以省略**，只有 key 和 effect 也是合法的 taint。
- 以 `node-role.kubernetes.io/master:NoSchedule` 為例：
  - **key**：`node-role.kubernetes.io/master`
  - **value**：沒有設定（空值）
  - **effect**：`NoSchedule`
- 這種寫法在 Kubernetes 預設 taint 很常見。

#### YAML 實例
```yaml
taints:
- effect: NoSchedule
  key: node-role.kubernetes.io/master
  value: ""
```

---

## 小結
- Taint 是「Node 的限制條件」，Toleration 是「Pod 的容忍條件」。
- 兩者搭配，可以靈活控制 Pod 的排程行為，讓叢集更安全、更有彈性。
