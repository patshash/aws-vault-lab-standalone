# Consumer Design: vault-ec2-sydney

**Branch**: 001-vault-ec2-sydney
**Date**: 2026-03-31
**Status**: Draft
**Provider**: aws ~> 5.0
**Terraform**: >= 1.14
**HCP Terraform Org**: pcarey-org

---

## Table of Contents

1. [Purpose & Requirements](#1-purpose--requirements)
2. [Module Selection & Architecture](#2-module-selection--architecture)
3. [Module Wiring](#3-module-wiring)
4. [Security Controls](#4-security-controls)
5. [Implementation Checklist](#5-implementation-checklist)
6. [Open Questions](#6-open-questions)

---

## 1. Purpose & Requirements

This consumer deployment is intended to provision an operator-accessible AWS lab in `ap-southeast-2` containing a dedicated Vault server, one RHEL host, and one Windows host. The RHEL and Windows hosts must run Vault Agent and trust a self-signed CA issued by the Vault server so they can authenticate over TLS while remaining reachable from the operator host for SSH, RDP, and Vault administration.

**Scope boundary**: This design excludes production-grade Vault HA, certificate lifecycle automation beyond the initial self-signed Vault CA, application deployment beyond Vault and Vault Agent bootstrap, and any implementation that requires raw infrastructure resources outside the consumer constitution.

### Requirements

**Functional requirements** -- what the deployment must provision (from Phase 1 clarification):

- Provision a new AWS network environment in `ap-southeast-2` for the lab deployment.
- Provision exactly three compute nodes: one Vault server, one RHEL host, and one Windows host.
- Use the Windows AMI lookup pattern `hc-base-windows-server-2025-x64-*` owned by `888995627335`.
- Install and configure Vault Agent on the RHEL and Windows hosts.
- Configure Vault server and client communication to use TLS with a self-signed CA issued by Vault.
- Allow operator access from the current public IP for SSH to Linux hosts, RDP to the Windows host, and Vault administration traffic as required.

**Non-functional requirements** -- constraints like compliance, performance, availability, cost:

- Compose infrastructure only from modules that are valid under the consumer constitution.
- Target HCP Terraform organization `pcarey-org`, project `ai-project`, and workspace `vault-ec2-sydney`.
- Use the shared variable set `aws-credential-doormat` rather than static AWS keys.
- Keep ingress least-privilege by restricting public administration access to the operator CIDR.
- Treat any bootstrap tokens, certificates, or rendered secrets as sensitive workspace values.
- Prefer a single-AZ lab-grade topology over a production HA footprint.

---

## 2. Module Selection & Architecture

### Architectural Decisions

**Private-registry-first design**: The design stays within the consumer workflow and does not invent raw resources or unsupported module interfaces.
*Rationale*: Registry discovery for `pcarey-org` found no private EC2 or Vault modules that can satisfy the compute and bootstrap requirements, so the design must document the blocker rather than fabricate a build path. See [research-private-registry-modules.md](/workspace/specs/001-vault-ec2-sydney/research-private-registry-modules.md).
*Rejected*: Assuming public modules or raw resources would violate the consumer constitution without an explicit exception.

**Workspace-aligned deployment**: The target runtime is HCP Terraform workspace `vault-ec2-sydney` in project `ai-project`.
*Rationale*: Org and project access are validated, the credentials pattern is already chosen, and the workspace model matches the repo's consumer workflow requirements. See [research-workspace-deployment.md](/workspace/specs/001-vault-ec2-sydney/research-workspace-deployment.md).
*Rejected*: Local execution and static AWS credentials were rejected because the consumer constitution requires HCP Terraform remote execution patterns.

**Operator-restricted access**: Administrative and Vault ingress should be restricted to the operator public IP, while host-to-host Vault traffic remains limited to the lab network.
*Rationale*: This satisfies the user's accessibility requirement without broadening exposure beyond what the lab needs. See [research-module-wiring.md](/workspace/specs/001-vault-ec2-sydney/research-module-wiring.md).
*Rejected*: Broad `0.0.0.0/0` ingress for SSH, RDP, or Vault was rejected as unnecessary.

**Vault-issued bootstrap trust**: The intended trust model is a self-signed CA issued by the Vault server and distributed to the RHEL and Windows clients during bootstrap.
*Rationale*: This matches the clarified security requirement and allows the clients to validate Vault over TLS without requiring ACM or a private CA. See [research-vault-bootstrap.md](/workspace/specs/001-vault-ec2-sydney/research-vault-bootstrap.md).
*Rejected*: ACM-backed certificates were rejected because the user selected Vault-issued self-signed trust material.

### Module Inventory

| Module | Registry Source | Version | Purpose | Conditional | Key Inputs | Key Outputs |
|--------|---------------|---------|---------|-------------|------------|-------------|
| Network module | TBD private AWS network module in `pcarey-org` | TBD | Create the VPC, subnets, and routing required for the lab | Blocked until available | region, CIDR, subnet layout | `vpc_id`, subnet IDs |
| Security module | TBD private AWS security-group module in `pcarey-org` | TBD | Create least-privilege ingress for SSH, RDP, and Vault | Blocked until available | `vpc_id`, operator CIDR, allowed ports | security group IDs |
| Vault server module | TBD private AWS compute or Vault module in `pcarey-org` | TBD | Provision the Vault server and emit client trust material if supported | Blocked until available | subnet ID, security groups, instance type, bootstrap hooks | private IP, admin address, CA artifact |
| Linux client module | TBD private AWS compute module in `pcarey-org` | TBD | Provision the RHEL host and install Vault Agent | Blocked until available | subnet ID, security groups, instance type, bootstrap hooks | private IP |
| Windows client module | TBD private AWS compute module in `pcarey-org` | TBD | Provision the Windows host with the approved 2025 AMI and install Vault Agent | Blocked until available | subnet ID, security groups, instance type, Windows AMI lookup, bootstrap hooks | private IP |

### Glue Resources

| Resource Type | Logical Name | Purpose | Depends On |
|---------------|-------------|---------|------------|
| -- | -- | No glue resources are approved at design time; the consumer path remains blocked on private module availability. | -- |

### Workspace Configuration

| Setting | Value | Notes |
|---------|-------|-------|
| Organization | `pcarey-org` | HCP Terraform organization |
| Project | `ai-project` | Target project for the workspace |
| Workspace | `vault-ec2-sydney` | Target workspace |
| Execution Mode | Remote | HCP Terraform managed |
| Terraform Version | `>= 1.14` | Matches repo and constitution expectations |
| Variable Sets | `aws-credential-doormat` | Shared AWS credential source |
| VCS Connection | `patshash/aws-vault-lab-standalone` on branch `001-vault-ec2-sydney` | Manual or VCS-driven workspace setup |

---

## 3. Module Wiring

### Wiring Diagram

```
network.vpc_id                ──→ security.vpc_id
network.public_subnet_ids[0]  ──→ vault_server.subnet_id
network.public_subnet_ids[1]  ──→ rhel_client.subnet_id
network.public_subnet_ids[2]  ──→ windows_client.subnet_id
security.admin_sg_id          ──→ vault_server.security_group_ids
security.admin_sg_id          ──→ rhel_client.security_group_ids
security.admin_sg_id          ──→ windows_client.security_group_ids
vault_server.private_ip       ──→ rhel_client.vault_addr
vault_server.private_ip       ──→ windows_client.vault_addr
vault_server.ca_artifact      ──→ rhel_client.vault_ca_bundle
vault_server.ca_artifact      ──→ windows_client.vault_ca_bundle
```

### Wiring Table

| Source Module | Output | Target Module | Input | Type | Transformation |
|--------------|--------|--------------|-------|------|----------------|
| Network module | `vpc_id` | Security module | `vpc_id` | string | direct |
| Network module | `public_subnet_ids[0]` | Vault server module | `subnet_id` | string | index selection |
| Network module | `public_subnet_ids[1]` | Linux client module | `subnet_id` | string | index selection |
| Network module | `public_subnet_ids[2]` | Windows client module | `subnet_id` | string | index selection |
| Security module | `admin_sg_id` | Vault server module | `security_group_ids` | list(string) | wrap in single-item list if required |
| Security module | `admin_sg_id` | Linux client module | `security_group_ids` | list(string) | wrap in single-item list if required |
| Security module | `admin_sg_id` | Windows client module | `security_group_ids` | list(string) | wrap in single-item list if required |
| Vault server module | `private_ip` | Linux client module | `vault_addr` | string | prefix with `https://` and port `8200` |
| Vault server module | `private_ip` | Windows client module | `vault_addr` | string | prefix with `https://` and port `8200` |
| Vault server module | `ca_artifact` | Linux client module | `vault_ca_bundle` | string | direct or decode, module-dependent |
| Vault server module | `ca_artifact` | Windows client module | `vault_ca_bundle` | string | direct or decode, module-dependent |

### Provider Configuration

```hcl
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "terraform"
      Owner       = var.owner
    }
  }

  # Dynamic credentials via HCP Terraform
  # assume_role { role_arn = var.deploy_role_arn }  # If the selected workspace uses cross-account access
}
```

### Variables

| Variable | Type | Required | Default | Validation | Sensitive | Description |
|----------|------|----------|---------|------------|-----------|-------------|
| `project_name` | string | Yes | -- | non-empty | No | Logical project name used in tags and naming |
| `environment` | string | Yes | `lab` | one of `lab`, `dev`, `sandbox` | No | Environment label for tags and workspace metadata |
| `owner` | string | Yes | -- | non-empty | No | Owner tag value |
| `aws_region` | string | Yes | `ap-southeast-2` | must equal `ap-southeast-2` for this design | No | AWS region for the deployment |
| `allowed_admin_cidr` | string | Yes | -- | must be a single IPv4 CIDR | No | Operator public IP range allowed to reach SSH, RDP, and Vault |
| `vault_instance_type` | string | Yes | -- | non-empty | No | Instance size for the Vault server |
| `rhel_instance_type` | string | Yes | -- | non-empty | No | Instance size for the RHEL host |
| `windows_instance_type` | string | Yes | -- | non-empty | No | Instance size for the Windows host |
| `windows_ami_owner` | string | No | `888995627335` | must be a 12-digit AWS account ID | No | Owner used for the Windows 2025 AMI lookup |
| `windows_ami_name_pattern` | string | No | `hc-base-windows-server-2025-x64-*` | non-empty | No | Name pattern used to discover the Windows AMI |
| `vault_bootstrap_secret` | string | Yes | -- | -- | Yes | Sensitive bootstrap material if the selected Vault module requires it |
| `vault_agent_version` | string | No | -- | semantic version or empty | No | Optional Vault Agent version override if exposed by the compute module |

### Outputs

| Output | Type | Source | Description |
|--------|------|--------|-------------|
| `vpc_id` | string | network module `vpc_id` | VPC identifier for the deployed lab |
| `vault_private_ip` | string | Vault server module private IP output | Private IP of the Vault server |
| `rhel_private_ip` | string | Linux client module private IP output | Private IP of the RHEL host |
| `windows_private_ip` | string | Windows client module private IP output | Private IP of the Windows host |
| `vault_addr` | string | derived from Vault server module | TLS address used by operators and Vault Agents |

---

## 4. Security Controls

| Control | Enforcement | Module Config | Reference |
|---------|-------------|---------------|-----------|
| Encryption at rest | Require module defaults or explicit encrypted root volumes for all instances | TBD in the approved compute modules | AWS Well-Architected Security Pillar: Data protection |
| Encryption in transit | Require Vault listener TLS and client trust of the Vault-issued CA | Vault server and client modules must expose TLS bootstrap inputs | AWS Well-Architected Security Pillar: Data protection in transit |
| Public access | Restrict `22/tcp`, `3389/tcp`, and `8200/tcp` to `var.allowed_admin_cidr`; avoid broader ingress | TBD in the approved security module | AWS Well-Architected Security Pillar: Network protection |
| IAM least privilege | Prefer instance profiles limited to Vault Agent auth or bootstrap requirements only | TBD in the approved compute modules | AWS Well-Architected Security Pillar: Identity management |
| Logging | Require OS and Vault logs to remain enabled where supported by the selected modules | TBD in the approved compute and Vault modules | AWS Well-Architected Security Pillar: Detective controls |
| Tagging | Enforce provider `default_tags` plus module-level tags for project, environment, owner, and managed-by metadata | provider `default_tags` and module `tags` inputs | AWS Well-Architected Security Pillar: Resource ownership |

---

## 5. Implementation Checklist

- [ ] **A: Confirm module sources** -- Identify approved private modules in `pcarey-org` for networking, security groups, Vault server compute, and generic EC2 compute before any implementation files are modified.
- [ ] **B: Scaffold consumer files** -- Create or update `versions.tf`, `backend.tf`, `providers.tf`, `variables.tf`, `locals.tf`, and `outputs.tf` for the `vault-ec2-sydney` workspace configuration and input contract.
- [ ] **C: Compose approved modules** -- Implement `main.tf` with the selected private modules, subnet placement, AMI lookup handling, and security-group wiring once the module inventory is resolved.
- [ ] **D: Configure bootstrap flows** -- Add the Vault server and Vault Agent bootstrap inputs, CA distribution wiring, and operator-CIDR access restrictions after the module interfaces are known.
- [ ] **E: Validate and document** -- Run formatting and validation, prepare example variable values, and update the README and issue with any implementation constraints discovered.

---

## 6. Open Questions

- Which private modules in `pcarey-org` should be used for VPC, security groups, Vault server compute, and generic EC2 compute? None are currently discoverable via the registry tools for the required compute and Vault capabilities.
- If no such private modules exist, should the workflow be allowed to consume public modules, or should those public modules first be wrapped or published privately?
- How should the Vault CA bundle be distributed to the RHEL and Windows hosts if the eventual Vault module does not emit a direct CA artifact?
- What exact operator public IP CIDR should be captured at implementation time for `allowed_admin_cidr`?

---
