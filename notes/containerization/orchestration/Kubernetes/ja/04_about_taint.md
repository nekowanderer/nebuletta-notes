# Taint（テイント）について

[English](../en/04_about_taint.md) | [繁體中文](../zh-tw/04_about_taint.md) | [日本語](../ja/04_about_taint.md) | [インデックスに戻る](../README.md)

---

## 1. マスターノードは Pod を実行できるか？

- **理論的には可能ですが、デフォルトでは推奨されておらず、許可もされていません。**
- マスターノードは主にクラスター管理（API サーバー、スケジューラー、コントローラマネージャなど）を担当し、一般的なアプリケーション Pod の実行には使われません。
- デフォルトで taint（例：`node-role.kubernetes.io/master:NoSchedule` や `node-role.kubernetes.io/control-plane:NoSchedule`）が付与されており、スケジューラーが一般 Pod をマスターノードに割り当てるのを防ぎます。
- taint を手動で削除すれば、マスターノードでも Pod を実行できます。これは minikube や k3s などのテスト環境でよく見られます。
- 本番環境では、マスターノードはコントロールコンポーネントのみを実行し、ワーカーノードがアプリケーション Pod を担当することが推奨されます。

---

## 2. Taint（テイント）の概念とは？

- **Taint（テイント）** とは、ノードに設定する制限条件であり、どの Pod がそのノードにスケジューリングできるかを制御します。
- **Toleration（トレランス）** は Pod 側に設定し、特定の taint を許容できることを示します。これにより、その taint を持つノードにスケジューリング可能となります。

### Taint のフォーマット
```
key=value:effect
```
- **key**：カスタム名
- **value**：カスタム値
- **effect**：効果タイプ
  - `NoSchedule`：対応する toleration がない Pod はこのノードにスケジューリングされません
  - `PreferNoSchedule`：できるだけスケジューリングしないが、絶対禁止ではありません
  - `NoExecute`：スケジューリングを防ぐだけでなく、既存の toleration を持たない Pod も退去（Evict）されます

### 実例
- マスターノードのデフォルト taint：
  ```
  node-role.kubernetes.io/master:NoSchedule
  ```
  対応する toleration がない Pod はこのノードにスケジューリングできません。

### Taint を使う場面
- マスターノードを保護し、一般アプリケーション Pod の実行を防ぐ場合
- 特殊なハードウェア（GPU など）を持つノードで、特定 Pod のみ許可したい場合
- リソース分離、メンテナンス、テストなど

---

### `node-role.kubernetes.io/master:NoSchedule` のフォーマットについて

- 標準フォーマットは `key=value:effect` ですが、**value は省略可能** です。key と effect だけでも有効な taint となります。
- 例：`node-role.kubernetes.io/master:NoSchedule`
  - **key**：`node-role.kubernetes.io/master`
  - **value**：未設定（空）
  - **effect**：`NoSchedule`
- この書き方は Kubernetes のデフォルト taint でよく使われます。

#### YAML 例
```yaml
taints:
- effect: NoSchedule
  key: node-role.kubernetes.io/master
  value: ""
```

---

## まとめ
- Taint は「ノード側の制限条件」、Toleration は「Pod 側の許容条件」です。
- 両者を組み合わせることで、Pod のスケジューリングを柔軟に制御でき、クラスターの安全性と柔軟性が向上します。 