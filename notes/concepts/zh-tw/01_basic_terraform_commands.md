# Terraform 基本指令

[English](../en/01_basic_terraform_commands.md) | [繁體中文](01_basic_terraform_commands.md) | [日本語](../ja/01_basic_terraform_commands.md) | [回到索引](../README.md)

### 指令
#### init
初始化指令對所有 Terraform 模組都是必需的：
```bash 
$ terraform init 
```
在初始化階段，Terraform 會下載提供者（provider）。一般來說，Terraform 核心只是一個抽象層；資源的具體行為和它們之間的交互由提供者決定。

#### plan
Plan 指令會嘗試比較所有 .tf 文件和公有雲上的實際內容。它會顯示差異並輸出關於變更的計劃。
```bash
$ terraform plan
```

#### apply
如果計劃符合我們的期望，意味著所有 .tf 文件都可以使用，那麼我們可以對公有雲運行 apply 指令來生成我們需要的資源。在輸入 `yes` 之前，請確保檢查所有應用資訊。
```bash
$ terraform apply
```

#### destroy
```bash
$ terraform destroy
```
Destroy 指令會移除當前 Terraform 配置管理的所有資源。本質上，它運行的是一個針對移除所有內容的計劃。Terraform 會顯示刪除計劃並在繼續前請求確認，類似於 apply 指令。在生產環境中使用時請特別小心。

#### 一般流程
`init` -> `plan` -> `apply` -> `destroy`（可選） 