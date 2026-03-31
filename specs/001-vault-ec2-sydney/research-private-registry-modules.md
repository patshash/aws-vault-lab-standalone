# Research: Private Registry Modules

## Question

Which private registry modules available in HCP Terraform org `pcarey-org` can compose this deployment?

## Findings

- `pcarey-org` currently exposes nine discoverable modules through the registry query tools.
- The modules that are clearly private are:
  - `app.terraform.io/pcarey-org/multicloud/dns`
  - `app.terraform.io/pcarey-org/nested/cloud`
  - `app.terraform.io/pcarey-org/onboarding/tfe`
  - `app.terraform.io/pcarey-org/route53-subdomain/aws`
  - `app.terraform.io/pcarey-org/static-app/github`
  - `app.terraform.io/pcarey-org/submodules/null`
- None of the discoverable private modules are suitable for provisioning a VPC, EC2 instances, Vault, or instance security groups for this workload.
- Registry searches for `ec2`, `instance`, and `vault` returned no private modules.
- Searches for `vpc` and `security` returned only public-registry namespaces:
  - `terraform-aws-modules/vpc/aws`
  - `terraform-aws-modules/security-group/aws`

## Assessment

- Under the consumer constitution, this workflow must compose infrastructure from private registry modules only.
- Based on the current registry inventory, the required building blocks for compute and Vault are not available as private modules in `pcarey-org`.
- The design can proceed only as a constrained plan that documents this gap and the decisions needed to unblock implementation.

## Implications For Design

- The design should treat module availability as the primary gating constraint.
- Viable next-step options are:
  - publish or grant access to private EC2 and Vault modules in `pcarey-org`
  - explicitly approve use of public modules despite the current consumer constitution
  - move the request to a module-authoring workflow instead of a consumer workflow
