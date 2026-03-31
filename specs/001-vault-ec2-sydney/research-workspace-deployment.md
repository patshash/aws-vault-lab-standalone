# Research: Workspace And Deployment

## Question

What HCP Terraform workspace and provider configuration should this consumer deployment use?

## Findings

- The organization `pcarey-org` is accessible with the configured token.
- The project `ai-project` exists in that organization.
- A workspace named `vault-ec2-sydney` does not yet exist and should be created in `ai-project`.
- The user specified `ap-southeast-2` and the shared variable set `aws-credential-doormat` for AWS access.
- The consumer constitution requires:
  - `cloud {}` backend configuration for HCP Terraform
  - remote execution
  - dynamic credentials or an approved workspace/project variable set pattern
  - no static AWS keys in code

## Recommended Workspace Configuration

| Setting | Value |
|---------|-------|
| Organization | `pcarey-org` |
| Project | `ai-project` |
| Workspace | `vault-ec2-sydney` |
| Region variable | `ap-southeast-2` |
| Execution mode | `remote` |
| Variable set | `aws-credential-doormat` |
| Provider constraint | `hashicorp/aws ~> 5.0` |
| Terraform version | `>= 1.14` |

## Variables To Document In Design

- `project_name`
- `environment`
- `owner`
- `aws_region`
- `allowed_admin_cidr`
- `vault_instance_type`
- `rhel_instance_type`
- `windows_instance_type`
- `windows_ami_owner`
- `windows_ami_name_pattern`
- any bootstrap or Vault Agent variables required by the eventual private modules

## Assessment

- Workspace configuration is not a blocker.
- The deployment path is valid once compatible private modules are available.
