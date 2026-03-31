# Research: Private Registry Modules

## Question

Which private registry modules available in HCP Terraform org `pcarey-org` can compose this deployment?

## Findings

- `pcarey-org` currently exposes nineteen discoverable modules through the registry query tools.
- The modules that are clearly private are:
  - `app.terraform.io/pcarey-org/multicloud/dns`
  - `app.terraform.io/pcarey-org/nested/cloud`
  - `app.terraform.io/pcarey-org/onboarding/tfe`
  - `app.terraform.io/pcarey-org/route53-subdomain/aws`
  - `app.terraform.io/pcarey-org/static-app/github`
  - `app.terraform.io/pcarey-org/submodules/null`
- None of the discoverable private modules are suitable for provisioning a VPC, EC2 instances, Vault, or instance security groups for this workload.
- `app.terraform.io/pcarey-org/route53-subdomain/aws` is the only discoverable private module directly relevant to DNS and hosted-zone integration for easier operator access.
- The registry metadata shows `route53-subdomain/aws` as a private no-code module, but the available tooling does not expose enough detail to confirm whether it manages individual record sets inside an existing hosted zone like `pcarey.sbx.hashidemos.io` or only handles delegated subdomains.
- The current registry inventory also exposes public AWS modules through the organization view, including:
  - `terraform-aws-modules/ec2-instance/aws`
  - `terraform-aws-modules/vpc/aws`
  - `terraform-aws-modules/security-group/aws`
  - `terraform-aws-modules/route53/aws`
  - `terraform-aws-modules/iam/aws`
- Of those, the most relevant candidates for this lab are `ec2-instance/aws`, `vpc/aws`, `security-group/aws`, and `route53/aws` if public-module usage is explicitly approved.
- Registry searches for `ec2`, `instance`, and `vault` returned no private modules.
- Searches for `rhel`, `windows`, and `vault` returned no private modules.
- Searches for `vpc`, `security`, and `instance` returned only public-registry namespaces:
  - `terraform-aws-modules/vpc/aws`
  - `terraform-aws-modules/security-group/aws`
  - `terraform-aws-modules/ec2-instance/aws`

## Hosted Zone Investigation

- The requested hosted zone is `pcarey.sbx.hashidemos.io`.
- Direct verification of the hosted zone from this shell session was not possible because local AWS Route 53 access is not available here.
- The design can still incorporate the hosted zone as an intended integration target, but implementation needs either:
  - confirmation that the zone already exists and is writable by the deployment credentials
  - or a module interface that handles delegation or record creation on behalf of the workspace

## Assessment

- Under the consumer constitution, this workflow must compose infrastructure from private registry modules only.
- Based on the current registry inventory, the required building blocks for compute and Vault are not available as private modules in `pcarey-org`.
- The design can proceed only as a constrained plan that documents this gap and the decisions needed to unblock implementation.

## Implications For Design

- The design should treat module availability as the primary gating constraint.
- DNS integration is feasible in principle because a private Route 53-related module exists, but its actual capabilities remain unverified.
- Viable next-step options are:
  - publish or grant access to private EC2 and Vault modules in `pcarey-org`
  - confirm whether `route53-subdomain/aws` can publish host records in `pcarey.sbx.hashidemos.io`
  - approve use of the publicly available `terraform-aws-modules` candidates surfaced in the org registry view, especially `ec2-instance/aws`, `vpc/aws`, `security-group/aws`, and `route53/aws`
  - explicitly approve use of public modules despite the current consumer constitution
  - move the request to a module-authoring workflow instead of a consumer workflow
