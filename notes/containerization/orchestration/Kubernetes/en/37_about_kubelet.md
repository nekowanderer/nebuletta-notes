# About kubelet

[English](../en/37_about_kubelet.md) | [繁體中文](../zh-tw/37_about_kubelet.md) | [日本語](../ja/37_about_kubelet.md) | [Back to Index](../README.md)

## What is kubelet?
`kubelet` is a **local daemon (agent)** that runs on every Kubernetes Node, responsible for communicating with the K8s API Server and **ensuring that Pods are properly executed on that Node**.

---

## Core Responsibilities
1. **Receive Pod Specifications (PodSpec)**  
   Receive Pod descriptions to be executed from the API Server.
2. **Start and Monitor Containers**  
   Usually execute through container runtime (such as Docker or containerd).
3. **Report Status**  
   Send back Pod and Node health status, resource usage information to the API Server.
4. **Health Checks**  
   Execute liveness/readiness probes to verify container health.
5. **Handle Lifecycle**  
   Including Pod startup, termination, restart behaviors.

---

## What kubelet is NOT responsible for
- Not responsible for scheduling (this is kube-scheduler's job)
- Does not manage other Nodes (it only manages the Node it runs on)

---

## Etymology
- `kubelet` = `kube` (Kubernetes) + `let` (small)
- `let` is a **diminutive** suffix in English
  - Common examples:
    - `piglet` (little pig)
    - `booklet` (small book)
    - `leaflet` (flyer)
- So `kubelet` means:
  > **"A small member of Kubernetes" or "little daemon"**

---

## Summary
| Item | Description |
|------|-------------|
| Function | Manage Pods on local Node |
| Communication Target | Interact with K8s API Server |
| Runtime Location | One runs on each Node |
| Name Meaning | Kubernetes' small unit daemon |

---

## Suggested Further Reading
- Interaction mechanism between kubelet and container runtime (Container Runtime Interface, CRI)
- Differences between kubelet and kube-proxy
- kubelet startup parameters and systemd management