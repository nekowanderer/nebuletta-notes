# Resource Mutation Caused by Terraform State Misconfiguration

[English](03_incorrect_shared_state_behavior.md) | [繁體中文](../zh-tw/03_incorrect_shared_state_behavior.md) | [日本語](../ja/03_incorrect_shared_state_behavior.md) | [Back to Index](../README.md)

---

## Background

- Date: 2025/05/24
- Difficulty: 🤬🤬
- Description: While managing multiple modules (`infra-networking` and `isolated-ec2`) with Terraform, an improper backend configuration led to overlapping state data and unintentional resource deletion.

---

## Observed Issue

- After successfully applying the `infra-networking` module, all expected resources (VPC, Subnets, Route Tables, etc.) were confirmed to exist in the AWS Console.
- When running `terraform plan` on the `isolated-ec2` module, Terraform attempted to remove all the resources created by `infra-networking`.
- This behavior deviates from the expected incremental layering and instead results in both modules operating within the same context.

---

## Architecture Diagram – Incorrect Setup

### ❌ Shared Terraform State Between infra-network and isolated-ec2

📁 state-storage/
└── dev-backend.hcl     --> Shared backend config

📁 infra-network/
└── main.tf             --> Uses dev-backend.hcl

📁 isolated-ec2/
└── main.tf             --> Also uses dev-backend.hcl

📦 AWS S3
└── key = dev/terraform.tfstate
    └── state contains resources from both infra and ec2

```
          +-----------------------------+
          |  S3: dev/terraform.tfstate  |
          +-----------------------------+
                  ▲          ▲
                  |          |
       +----------+          +----------+
       |                                |
+---------------+             +------------------+
| infra-network |             |  isolated-ec2    |
+---------------+             +------------------+
(Shared state, mutually affecting each other)
```

---

## Recommended Setup

### ✅ Separate Terraform State for infra-network and isolated-ec2

📁 state-storage/
├── dev-infra.hcl       --> For infra-network
└── dev-ec2.hcl         --> For isolated-ec2

📁 infra-network/
└── main.tf             --> Uses dev-infra.hcl

📁 isolated-ec2/
└── main.tf             --> Uses dev-ec2.hcl

📦 AWS S3
├── key = dev/infra/terraform.tfstate
└── key = dev/ec2/terraform.tfstate

          +---------------------------------+       +--------------------------------+
          | S3: dev/infra/terraform.tfstate |       | S3: dev/ec2/terraform.tfstate  |
          +---------------------------------+       +--------------------------------+
                           ▲                                        ▲
                           |                                        |
                   +---------------+                       +------------------+
                   | infra-network |                       |  isolated-ec2    |
                   +---------------+                       +------------------+
                                (Logically isolated)

---

## Debugging Process

- Initially, only `infra-networking` had its backend properly configured and its state successfully uploaded to S3.
- `isolated-ec2` lacked its own backend configuration, resulting in Terraform falling back to default or reusing `dev-backend.hcl`.
- Terraform assumed that the entire state belonged to `isolated-ec2`, and thus attempted to "synchronize" it, leading to deletion of the infra resources.

---

## Resolution

1. Create a dedicated backend configuration file (HCL) for `isolated-ec2`.
2. Use a script to automatically generate backend configs based on environment and module naming.
3. Ensure each backend config has a unique `key` and DynamoDB lock table to prevent conflicts.
4. Re-run `terraform init` → `plan` with the corrected backend; resource deletion issue no longer occurs.

---

## Additional Notes

In this case, the backend configuration is organized into per-environment and per-module folders, and backend HCL files are automatically generated using a shell script.

Documentation for the backend generation script can be found in the `terraform/envs` folder ([README](../../../envs/README.md)).

---

## Summary

If multiple Terraform modules share the same backend without proper segregation, it may lead to unexpected deletions or resource drift.

**Key takeaway**: Proper segregation and management of state files is essential for building reliable multi-module infrastructure.
