# Terraform Standards

## Overview

These standards define baseline Terraform practices for infrastructure as code. They focus on consistent structure, safe state management, and repeatable workflows across environments and teams. These are Terraform standards, not tool-specific instructions. Tools like Terraflow may implement these conventions, but the standards apply to any Terraform usage.

## Project Structure

### Root Terraform Directory Layout

Use a consistent root layout to improve readability and reduce cognitive load:

```
terraform/
├── _init.tf       # Provider + backend configuration
├── inputs.tf      # input variables
├── locals.tf      # local values and derived config
├── main.tf        # resources and module calls
├── outputs.tf     # outputs
```

#### Standards
- Use `_init.tf` (or a similar file name) for provider and backend configuration.
- Keep intent clear: prefer a stable, predictable layout so dependencies are easy to follow.
- Avoid mixing responsibilities (e.g., variables scattered across multiple files without a pattern).

### Module Structure

```
terraform/
└── modules/
    ├── networking/
    │   ├── inputs.tf
    │   ├── main.tf
    │   └── outputs.tf
    └── compute/
        ├── inputs.tf
        ├── main.tf
        └── outputs.tf
```

#### Standards
- Use `terraform/modules/` for reusable components.
- Each module uses `inputs.tf`, `main.tf`, `outputs.tf`.
- Prefer reusable modules to keep configurations DRY across environments or projects.

## Backend Configuration

### Standards
- Do not hardcode backend settings directly in `.tf` files.
- Use `-backend-config` flags or separate backend config files.
- Prefer local backend for local development; remote backend for shared or CI environments.
- Store state in the same cloud account/region as the resources being provisioned.
- Use environment variables or templating for backend values (bucket, key, region).

Note: Terraflow can supply backend config and templating, but any workflow can implement this using Terraform's native `-backend-config` support.

## Workspaces

### Standards
- Use Terraform workspaces for environment isolation (e.g., dev, staging, prod).
- Workspace selection should be explicit and automated where possible.

### Automated Workspace Selection (Hierarchy)

When workspace selection is automated (CI/CD, wrappers, scripts), use this hierarchy (first match wins):

1. CLI override: explicit `--workspace` or equivalent
2. Environment variable: `TF_WORKSPACE` or `TERRAFLOW_WORKSPACE`
3. Git tag: if checked out on a tag (e.g. `v1.0.0` -> `v1-0-0`)
4. Git branch: if non-ephemeral (e.g. `main` -> `main`)
5. Hostname: fallback when no git repo, no tag, or ephemeral branch

#### Ephemeral Branches
- Branches matching `^[^/]+/` (e.g., `feature/foo`, `fix/bar`) are ephemeral.
- Do not use ephemeral branch names as workspace names.
- Use a shared dev workspace or hostname fallback instead.

#### Workspace Naming Rules
- Must be valid Terraform identifiers: alphanumeric, underscores, hyphens.
- Avoid provider-specific invalid characters.
- Keep names stable and deterministic.

Note: Terraflow uses this hierarchy, but any CI/CD wrapper can implement it.

## Provider and Terraform Versions

### Standards
- Pin Terraform version using `required_version` (minimum `>= 1.0` or stricter).
- Pin provider versions using constraints (e.g., `~> 5.0`).
- Use `required_providers` in the root module.

```hcl
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

## Configuration and Variables

### Standards
- Prefer convention over configuration with sensible defaults.
- Use environment variables (`TF_VAR_*`) for sensitive or environment-specific values.
- Avoid hardcoding regions, account IDs, or repository names; use variables or data sources.

## Resource Tagging

### Standards
- Tag all resources with:
  - `workspace`
  - `managed-by`
  - `repository`
  - `commit` or `version` (when available)
- Use `terraform.workspace` for workspace identification in tags.
- For GCP, use lowercase, hyphenated names where required by the provider.

```hcl
tags = {
  workspace  = terraform.workspace
  managed-by = "terraform"
  repository = var.repository
  commit     = var.commit_sha
}
```

## Design Principles

### Standards
- Support multiple clouds (AWS, Azure, GCP) with consistent patterns.
- Use workspaces for preview/feature environments.
- Keep application code and Terraform in the same repository when practical.
- Align with Twelve-Factor principles (config via env, backing services, etc.).

## References

- Git Workflow Standards (workspace naming and environment strategy)
- AWS Architecture Standards (provider requirements and examples)
- Deployment Standards (Infrastructure as Code principles)
