# kubeletについて

[English](../en/37_about_kubelet.md) | [繁體中文](../zh-tw/37_about_kubelet.md) | [日本語](../ja/37_about_kubelet.md) | [インデックスに戻る](../README.md)

## kubeletとは？
`kubelet`は、各Kubernetes Node上で動作する**ローカルデーモン（エージェント）**で、K8s API Serverとの通信を担当し、**そのNode上でPodが正しく実行されることを保証**します。

---

## 主な責務
1. **Pod定義（PodSpec）の受信**  
   API Serverから実行すべきPod記述を受け取る。
2. **コンテナの起動と監視**  
   通常はcontainer runtime（Dockerやcontainerdなど）を通じて実行。
3. **ステータスの報告**  
   PodとNodeの健康状態、リソース使用状況をAPI Serverに送り返す。
4. **ヘルスチェック**  
   liveness/readiness probesを実行し、コンテナの健康状態を確認。
5. **ライフサイクルの処理**  
   Podの起動、終了、再起動などの動作を含む。

---

## kubeletが担当しないこと
- スケジューリングは担当しない（これはkube-schedulerの仕事）
- 他のNodeの管理はしない（自分が動作するNodeのみを管理）

---

## 名前の由来
- `kubelet` = `kube`（Kubernetes）+ `let`（小さい）
- `let`は英語の指小辞（diminutive）接尾辞
  - よくある例：
    - `piglet`（子豚）
    - `booklet`（小冊子）
    - `leaflet`（チラシ）
- したがって`kubelet`の意味は：
  > **「Kubernetesの小さなメンバー」または「小さなデーモン」**

---

## まとめ
| 項目 | 説明 |
|------|------|
| 機能 | ローカルNode上のPodを管理 |
| 通信対象 | K8s API Serverと相互作用 |
| 実行場所 | 各Nodeに1つずつ動作 |
| 名前の意味 | Kubernetesの小さな単位デーモン |

---

## 推奨参考資料
- kubeletとcontainer runtimeの相互作用メカニズム（Container Runtime Interface, CRI）
- kubeletとkube-proxyの違い
- kubeletの起動パラメータとsystemd管理方法