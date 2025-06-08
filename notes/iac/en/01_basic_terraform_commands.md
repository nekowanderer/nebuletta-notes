# Basic Terraform Commands

[English](01_basic_terraform_commands.md) | [繁體中文](../zh-tw/01_basic_terraform_commands.md) | [日本語](../ja/01_basic_terraform_commands.md) | [Back to Index](../README.md)

### Commands
#### init
Init command is necessary for all terraform modules:
```bash 
$ terraform init 
```
In init phase, terraform will download the provider, generally speaking, Terraform core is just an abstraction; the specific behavior of resources and their interactions are determined by the provider.

#### plan
Plan command will try to compare all the .tf files and the actual content on the public cloud. It will show up the differences and output the plan for about the changes.
```bash
$ terraform plan
```

#### apply
If the plan meet our expectation, which means all the .tf files are good to go, then we can run the apply command against the public cloud for generating the resources we need. Ensure that you review all the apply information before type `yes`.
```bash
$ terraform apply
```

#### output

```bash
$ terraform output
```

#### destroy
```bash
$ terraform destroy
```
Destroy command will remove all resources managed by the current Terraform configuration. It's essentially running a plan that targets the removal of everything. Terraform will show a deletion plan and ask for confirmation before proceeding, similar to the apply command. Use this with caution, especially in production environments.

#### General process
`init` -> `plan` -> `apply` -> `destroy`(optional)
