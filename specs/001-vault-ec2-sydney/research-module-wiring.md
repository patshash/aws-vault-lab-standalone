# Research: Module Wiring

## Question

How should outputs from networking modules wire into compute modules for one Vault server, one RHEL server, and one Windows host while restricting administration and Vault access to the operator IP?

## Findings

- The intended wiring pattern is conventional consumer composition:
  - VPC module outputs `vpc_id`, subnet IDs, route table IDs, and possibly CIDR blocks.
  - Security-group module inputs consume `vpc_id` and `allowed_cidr_blocks`.
  - Compute modules consume `subnet_id`, one or more security group IDs, instance profile data, and AMI selection inputs.
- For this workload, the Windows and RHEL instances should each be in a subnet reachable from the operator IP for SSH or RDP and for Vault client traffic.
- The Windows host must use an AMI lookup equivalent to the provided `aws_ami` data source pattern for `hc-base-windows-server-2025-x64-*` owned by `888995627335`.
- The Vault host and client hosts need bootstrap hooks so they can:
  - install Vault or Vault Agent
  - receive the Vault CA certificate
  - render agent configuration
  - start the service with TLS-enabled listener or client configuration

## Constraints

- Exact input and output names cannot be verified because no discoverable private EC2 or Vault modules are currently available in `pcarey-org`.
- If the eventual compute module does not expose `user_data`, `user_data_base64`, or equivalent bootstrap variables, this design cannot satisfy the Vault Agent installation requirement.
- If the eventual module does not allow custom AMI selection, the Windows 2025 image requirement cannot be satisfied.

## Expected Wiring Shape

```text
module.network.vpc_id              -> module.admin_security_group.vpc_id
module.network.public_subnet_ids   -> module.vault_server.subnet_id
module.network.public_subnet_ids   -> module.rhel_client.subnet_id
module.network.public_subnet_ids   -> module.windows_client.subnet_id
module.admin_security_group.id     -> module.vault_server.security_group_ids
module.admin_security_group.id     -> module.rhel_client.security_group_ids
module.admin_security_group.id     -> module.windows_client.security_group_ids
module.vault_server.private_ip     -> module.rhel_client.vault_addr
module.vault_server.private_ip     -> module.windows_client.vault_addr
module.vault_server.ca_artifact    -> module.rhel_client.vault_ca_bundle
module.vault_server.ca_artifact    -> module.windows_client.vault_ca_bundle
```

## Assessment

- The data flow is straightforward if suitable private modules exist.
- The blocking factor is not wiring complexity; it is the absence of verified private modules that expose the necessary AMI and bootstrap controls.
