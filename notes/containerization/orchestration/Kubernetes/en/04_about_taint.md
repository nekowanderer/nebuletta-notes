# About Taint

[English](../en/04_about_taint.md) | [繁體中文](../zh-tw/04_about_taint.md) | [日本語](../ja/04_about_taint.md) | [Back to Index](../README.md)

---

## 1. Can the Master Node Run Pods?

- **In theory, yes, but by default it is not recommended and not allowed.**
- The Master Node is mainly responsible for cluster management (API Server, Scheduler, Controller Manager, etc.), not for running general application Pods.
- By default, a taint (such as `node-role.kubernetes.io/master:NoSchedule` or `node-role.kubernetes.io/control-plane:NoSchedule`) is added to prevent the Scheduler from assigning general Pods to the Master Node.
- If you manually remove the taint, the Master Node can run Pods. This is common in test environments (such as minikube, k3s).
- In production, it is recommended that the Master Node only runs control components, and Worker Nodes run application Pods.

---

## 2. What is the Concept of Taint?

- **Taint** is a restriction set on a Node to control which Pods can be scheduled onto that Node.
- **Toleration** is set on a Pod, indicating that the Pod can tolerate a certain taint and thus can be scheduled onto a Node with that taint.

### Taint Format
```
key=value:effect
```
- **key**: Custom name
- **value**: Custom value
- **effect**: The effect type
  - `NoSchedule`: Pods without the corresponding toleration will not be scheduled onto this Node
  - `PreferNoSchedule`: Prefer not to schedule, but not strictly forbidden
  - `NoExecute`: Not only prevents scheduling, but also evicts existing Pods without toleration from this Node

### Example
- Default taint on Master Node:
  ```
  node-role.kubernetes.io/master:NoSchedule
  ```
  Pods without the corresponding toleration cannot be scheduled onto this Node.

### When to Use Taint?
- To protect the Master Node from running general application Pods
- For Nodes with special hardware (such as GPU), only allow specific Pods
- For resource isolation, maintenance, testing, etc.

---

### About the Format `node-role.kubernetes.io/master:NoSchedule`

- Although the standard format is `key=value:effect`, **the value can be omitted**. Having only key and effect is also a valid taint.
- For example, `node-role.kubernetes.io/master:NoSchedule`:
  - **key**: `node-role.kubernetes.io/master`
  - **value**: not set (empty)
  - **effect**: `NoSchedule`
- This style is common in Kubernetes default taints.

#### YAML Example
```yaml
taints:
- effect: NoSchedule
  key: node-role.kubernetes.io/master
  value: ""
```

---

## Summary
- Taint is a "restriction on the Node", Toleration is a "tolerance on the Pod".
- Using both together allows flexible control of Pod scheduling, making the cluster safer and more flexible. 