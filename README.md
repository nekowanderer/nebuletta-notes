# nebuletta-notes

**Nebuletta** is an experimental Infrastructure-as-Code (IaC) project aimed at rapidly provisioning self-managed cloud infrastructure using [Terraform](https://www.terraform.io/) on public cloud platforms.

This project is built with **AWS** as the default cloud provider. It includes a set of composable, self-contained modules that can be flexibly assembled based on different deployment scenarios.

To manage orchestration, **[Terramate](https://terramate.io/)** is used instead of introducing additional domain-specific frameworks like Terragrunt. This choice keeps the entire infrastructure stack purely in Terraform's native language, ensuring simplicity and consistency.
  
The implementation is maintained in a separate repository, [click here for the implementation](https://github.com/nekowanderer/nebuletta).

## Project Structure

```
nebuletta-notes/
├── notes/                   
│   ├── aws/                 # Notes for AWS related topics
│   ├── iac/                 # Notes for Infrastrucuture as Code  
│   ├── containerization/    # Notes for containerization (TBC)
│   ├── ai/                  # Notes for AI related topics (TBC)
│   └── troubleshooting/     # Experimentation and debugging notes
└── terraform/               # Terraform related files
    └── modules/             # Reusable Terraform modules
        ├── compute/         # Compute related modules
        ├── networking/      # Networking related modules
        ├── .../             # Other modules
        └── state-storage/   # State storage related modules
```
