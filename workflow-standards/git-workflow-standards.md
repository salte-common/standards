# Git Workflow Standards

## Overview

This document defines the standards for GitHub-based development workflows, release processes, and deployment automation. The workflow supports both simple continuous deployment scenarios and complex multi-environment release cycles with change management integration.

> **Note**: For comprehensive deployment and infrastructure standards, see [Deployment Standards](./../deployment-standards/deployment-standards.md). For operational procedures and monitoring standards, see [Operations Standards](./../operations-standards/operations-standards.md).

## Core Workflow Principles

### 1. Tag-driven Deployments
- **Tags trigger deployments**: Environment deployments are triggered by tags, not branch merges
- **Commit SHA traceability**: ServiceNow change requests reference specific commit SHAs for full traceability through environments
- **Permanent audit trail**: Production tags are never deleted and serve as permanent records

### 2. Branch Lifecycle Management
- **Transient vs. persistent branches**: Clear distinction between branches that live forever (main) and those created for specific purposes then deleted (feature, fix, release)
- **Branch isolation**: Release branches provide isolation for QA fixes without impacting ongoing development
- **Clean branch management**: Transient branches are deleted immediately after merge or deployment completion

### 3. Versioning Flexibility
- **Teams choose versioning convention**: Projects select semantic versioning or date-based versioning based on their needs
- **Consistent conventions**: Once chosen, the versioning convention must remain consistent throughout the project's lifecycle
- **Documentation requirement**: Versioning convention must be documented in the project's README

### 4. Infrastructure State Isolation
- **Terraform workspaces derived from tags**: Infrastructure state isolation achieved through workspace naming derived from tags and branches
- **Environment-specific workspaces**: Official environments use static workspace names, isolated environments use tag-derived names
- **State management**: Workspace lifecycle aligns with environment lifecycle

## Branch Strategy

### Persistent Branches

#### Main Branch

- **Name:** `main`
- **Purpose:** Primary development branch containing the latest integrated code
- **Protection Rules:**
  - Requires pull request with 1 approval from developer who is not a committer
  - Requires status checks to pass (static analysis, unit tests, linting, security scanning, build validation, terraform plan)
  - No direct pushes allowed
- **Deployment:** Automatically deploys to Development environment on push
- **Terraform Plan:** Feature branches plan against `main` workspace to show impact on development environment

### Transient Branches

All transient branches are created for a specific purpose and deleted after that purpose is complete.

#### Feature Branches

- **Naming Convention:** `feature/<feature-name>`
- **Purpose:** Development of new features or fixes to non-production code
- **Lifecycle:** 
  1. Created from `main` or `release/*` branch
  2. Development work committed to feature branch
  3. Pull request opened to source branch (`main` or `release/*`)
  4. Requires 1 approval from developer who is not a committer
  5. Merged to source branch via pull request
  6. Deleted after merge
- **CI/CD on Push/PR:**
  - Static code analysis
  - Unit tests
  - Linting
  - Security scanning
  - Build validation
  - Terraform plan:
    - On push: plan against `main` workspace
    - On PR to main: plan against `main` workspace
    - On PR to release branch: plan against `qa` workspace
- **Optional:** Can create `preview-<feature-name>` tag for isolated feature testing environment

#### Fix Branches

- **Naming Convention:** `fix/<fix-name>`
- **Purpose:** Production hotfixes only (emergency fixes to production environment)
- **Important:** Use `feature/*` branches for fixes to development or QA code. Only use `fix/*` for production hotfixes.
- **Lifecycle:**
  1. Created from production tag (e.g., `prod-v1.2.3`)
  2. Minimal fix implemented and committed
  3. Tested in isolated dev environment via `dev-v*` tag
  4. Tagged for production deployment
  5. Merged back to `main` via pull request
  6. Deleted after merge
- **CI/CD on Push/PR:**
  - Static code analysis
  - Unit tests
  - Linting
  - Security scanning
  - Build validation
  - Terraform plan against `prod` workspace (shows impact on production)

#### Release Branches

- **Naming Convention:**
  - Semantic versioning: `release/X.Y.0` (e.g., `release/1.2.0`)
  - Date-based versioning: `release/YYYYMMDD` (e.g., `release/20241215`)
- **Purpose:** 
  - Stabilize code for release
  - Apply QA fixes without impacting ongoing development
  - Provide isolated environment for release-specific changes
- **When to Create:**
  - **Always created when promoting to QA** (standard practice, not optional)
  - Provides ability to apply fixes if issues found in QA
  - Can be used to redeploy to dev for testing fixes before returning to QA
- **Lifecycle:**
  1. Created from `main` when ready to promote to QA
  2. Tagged for QA deployment (e.g., `qa-v1.2.0`)
  3. QA fixes committed to release branch via pull requests
  4. **Fixes should be merged back to `main` as soon as possible** (developer may validate fix first before merging)
  5. Release branch retagged as needed for QA redeployment
  6. Tagged for production deployment after QA approval
  7. Final merge to `main` after successful production deployment (if not already merged)
  8. **Deleted after merge to main**
- **Protection Rules:**
  - Pattern: `release/*` (applies to all release branches)
  - Requires pull request with 1 approval from developer who is not a committer
  - Requires status checks to pass
  - No direct pushes allowed
- **CI/CD on Push/PR:**
  - Static code analysis
  - Unit tests
  - Linting
  - Security scanning
  - Build validation
  - Terraform plan against `qa` workspace

## Tagging Conventions

### Overview

Teams must choose **one** tagging convention per project and document the choice in the project's README. The convention should remain consistent throughout the project's lifecycle.

### Convention 1: Semantic Versioning

**Best for:**
- Libraries and APIs
- Products with defined compatibility contracts
- Projects where breaking changes need to be communicated clearly
- External-facing services with consumers who depend on version compatibility

**Format:** `{environment}-vX.Y.Z`

- **X** = Major version (breaking changes, incompatible API changes)
- **Y** = Minor version (new features, backwards compatible)
- **Z** = Patch version (bugfixes, backwards compatible)

**Tag Prefixes:**
- `preview-<feature-name>` - Isolated preview environment for feature testing
- `dev-vX.Y.Z` - Isolated development deployment (when needed from release branch)
- `qa-vX.Y.Z` - QA environment deployment
- `prod-vX.Y.Z` - Production environment deployment

**Examples:**

```
preview-user-authentication
dev-v1.2.0
qa-v1.2.0
qa-v1.2.1  (after QA bugfix)
prod-v1.2.1
```

**Version Incrementing:**
- Major version: Breaking API changes, major architectural changes
- Minor version: New features, new endpoints, backwards-compatible changes
- Patch version: Bugfixes, security patches, performance improvements

### Convention 2: Date-based Versioning

**Best for:**
- Internal applications
- Continuous delivery workflows
- Sprint-based or time-boxed releases
- Projects where delivery date is more important than semantic meaning

**Format:** `{environment}-vYYYYMMDD-N`

- **YYYYMMDD** = Target release date
- **N** = Sequential iteration number starting at 0 (increments for fixes applied to same release)

**Tag Prefixes:**
- `preview-<feature-name>` - Isolated preview environment for feature testing
- `dev-vYYYYMMDD-N` - Isolated development deployment (when needed from release branch)
- `qa-vYYYYMMDD-N` - QA environment deployment
- `prod-vYYYYMMDD-N` - Production environment deployment

**Examples:**

```
preview-new-dashboard
dev-v20241215-0
qa-v20241215-0
qa-v20241215-1  (after QA bugfix)
prod-v20241215-1
```

**Version Incrementing:**
- Date changes: New release targeting a different date
- N increments: Fixes or updates to the same release (0 → 1 → 2...)

### Tag Lifecycle and Retention

| Tag Type | When Created | When Deleted | Purpose |
|----------|--------------|--------------|---------|
| `preview-*` | From feature branch for isolated testing | After developer finishes testing (manual destroy) | Test features in isolation before merging |
| `dev-v*` | From release/fix branch for isolated dev testing | After developer finishes testing (manual destroy) | Test fixes or release branch changes in dev |
| `qa-v*` | From release branch when ready for QA | After production deployment complete | QA testing and validation |
| `prod-v*` | From release/fix branch when deploying to production | **NEVER** (permanent audit trail) | Production deployments, permanent record |

**Important Notes:**
- Production tags are **never deleted** and serve as permanent audit trail
- Preview and dev-v tags create isolated environments that should be manually destroyed when testing is complete
- QA tags should be deleted after successful production deployment to keep repository clean
- Multiple QA tags may exist for a single release if fixes are applied (qa-v1.2.0, qa-v1.2.1, etc.)

## Workflow Triggers

### Branch-based Workflows (Testing & Validation)

**Trigger:**
- Push to any branch
- Pull request to main or release branches

**Actions Performed:**
- Static code analysis
- Unit tests
- Linting
- Security scanning
- Build validation
- Terraform plan against target workspace:
  - Feature branches on push: plan against `main` workspace
  - Feature branches on PR: plan against target branch workspace (`main` or `qa`)
  - Release branches: plan against `qa` workspace
  - Fix branches: plan against `prod` workspace

**No Deployments** - Branch pushes and pull requests run quality checks only, except for main branch which auto-deploys to development.

**Example Workflow Configuration:**

```yaml
on:
  push:
    branches:
      - '**'
  pull_request:
    branches:
      - main
      - 'release/**'

jobs:
  quality-checks:
    steps:
      - name: Determine terraform workspace
        run: |
          # For push events
          if [[ "${{ github.event_name }}" == "push" ]]; then
            if [[ "${{ github.ref }}" == refs/heads/feature/* ]]; then
              WORKSPACE="main"
            elif [[ "${{ github.ref }}" == refs/heads/release/* ]]; then
              WORKSPACE="qa"
            elif [[ "${{ github.ref }}" == refs/heads/fix/* ]]; then
              WORKSPACE="prod"
            fi
          # For pull request events
          elif [[ "${{ github.event_name }}" == "pull_request" ]]; then
            if [[ "${{ github.base_ref }}" == "main" ]]; then
              WORKSPACE="main"
            elif [[ "${{ github.base_ref }}" == release/* ]]; then
              WORKSPACE="qa"
            fi
          fi
      - Static analysis
      - Unit tests
      - Linting
      - Security scanning
      - Build validation
      - Terraform plan (using determined workspace)
```

### Tag-based Workflows (Deployments)

**Trigger:**
- Push tags matching environment patterns:
  - `preview-*`
  - `dev-v*`
  - `qa-v*`
  - `prod-v*`

**Actions Performed:**
- Build artifacts
- Deploy to target environment
- Run smoke tests

**For Preview and Dev-v Tags:**
- Include manual destroy job for cleanup

**Example Workflow Configuration:**

```yaml
on:
  push:
    tags:
      - 'preview-*'
      - 'dev-v*'
      - 'qa-v*'
      - 'prod-v*'

jobs:
  deploy:
    steps:
      - Build artifacts
      - Deploy to environment
      - Run smoke tests
  
  destroy:
    if: manual trigger
    steps:
      - Terraform destroy
      - Delete workspace
```

### Main Branch Workflow (Auto-deploy to Development)

**Trigger:**
- Push to main branch

**Actions Performed:**
- Build artifacts
- Deploy to development environment (workspace: `main`)
- Run smoke tests

**Example Workflow Configuration:**

```yaml
on:
  push:
    branches:
      - main

jobs:
  deploy-dev:
    steps:
      - Build artifacts
      - Deploy to development
      - Run smoke tests
```

## Environment Strategy

### Environment Overview

| Environment | Trigger | Purpose | Workspace | Persistence |
|------------|---------|---------|-----------|-------------|
| **Preview** | Tag `preview-*` from feature branch | Developer testing, isolated feature demos | `preview-feature-name` | Ephemeral (manual destroy) |
| **Development** | Push to `main` branch | Integration testing, continuous deployment of latest code | `main` | Persistent |
| **Dev-isolated** | Tag `dev-v*` from release/fix branch | Isolated testing of fixes before QA/prod | `dev-vX-Y-Z` | Ephemeral (manual destroy) |
| **QA** | Tag `qa-v*` from release branch | Quality assurance, user acceptance testing | `qa` | Temporary (overwritten with new QA tags, deleted after prod deploy) |
| **Production** | Tag `prod-v*` from release/fix branch | Live production system | `prod` | Persistent |

### Environment Details

#### Preview Environments
- **Purpose:** Isolated testing of individual features before merging to main
- **Creation:** Developer tags feature branch with `preview-<feature-name>`
- **Isolation:** Each preview tag creates a completely separate environment with unique resources
- **Cleanup:** Developer manually triggers destroy job when testing complete
- **Access:** Development team only

#### Development Environment
- **Purpose:** Integration testing of latest merged code
- **Deployment:** Automatic on every merge to main
- **Stability:** Expected to be less stable, contains latest integrated features
- **Data:** Synthetic test data only
- **Workspace:** Always uses `main` workspace

#### Dev-isolated Environments
- **Purpose:** Test fixes from release or fix branches in development without impacting official dev environment
- **Creation:** Tag release/fix branch with `dev-vX.Y.Z` when isolated testing needed
- **Isolation:** Creates separate environment parallel to official dev
- **Cleanup:** Developer manually triggers destroy job when testing complete
- **Common Use Case:** Testing hotfixes before production deployment

#### QA Environment
- **Purpose:** Quality assurance testing and user acceptance testing
- **Deployment:** Manual tag creation from release branch
- **Stability:** Stable during QA cycle, only changes when new QA tag created
- **Data:** Sanitized production-like data
- **Workspace:** Uses `qa` workspace (overwritten with each new QA deployment)
- **Duration:** Exists until production deployment, then QA tags are deleted

#### Production Environment
- **Purpose:** Live production system serving end users
- **Deployment:** Requires approved ServiceNow Change Request
- **Stability:** Highly stable, change-controlled
- **Data:** Live production data
- **Workspace:** Uses `prod` workspace (persistent, never deleted)
- **Rollback:** Supported by deploying previous production tag

## Release Process

> **Note**: For comprehensive deployment standards including Infrastructure as Code and container deployment, see [Deployment Standards](./../deployment-standards/deployment-standards.md).

### Standard Release Flow

#### Step 1: Feature Development

```bash
# Create feature branch from main
git checkout main
git pull origin main
git checkout -b feature/my-feature

# Develop and commit changes
git add .
git commit -m "feat: implement my feature"
git push origin feature/my-feature
```

**Optional - Create Preview Environment:**

```bash
# Tag for isolated preview testing
git tag preview-my-feature
git push origin preview-my-feature

# Preview environment automatically deploys
# Test feature in isolation
# Manually destroy when done testing
```

#### Step 2: Merge to Main

```bash
# Create pull request to main
# Requires 1 approval from developer who is not a committer
# All quality checks must pass

# After approval and merge:
# - Main branch automatically deploys to Development environment
# - Integration testing happens in dev
# - Feature branch is deleted
```

#### Step 3: Prepare for QA

**Always create a release branch when promoting to QA:**

```bash
# Create release branch from main
git checkout main
git pull origin main
git checkout -b release/1.2.0  # or release/20241215

# Tag for QA deployment
git tag qa-v1.2.0  # or qa-v20241215-0
git push origin release/1.2.0 qa-v1.2.0

# QA environment automatically deploys
```

#### Step 4: QA Cycle

**If QA finds issues:**

```bash
# Create fix branch from release branch
git checkout release/1.2.0
git checkout -b feature/qa-fix

# Implement fix
git commit -m "fix: resolve QA issue"

# Create pull request to release branch
# Requires 1 approval from developer who is not a committer
# Merge fix to release branch

# Merge fix back to main as soon as possible
# (Developer may validate fix first via dev-v tag if needed)
git checkout main
git merge release/1.2.0  # or create PR from release to main
git push origin main

# Retag for QA redeployment (increment patch/iteration)
git checkout release/1.2.0
git pull origin release/1.2.0
git tag qa-v1.2.1  # or qa-v20241215-1
git push origin qa-v1.2.1

# QA tests the new deployment
# Repeat until QA approves
```

**Optional - Test Fix in Dev Before QA:**

```bash
# If you want to test fix in dev before deploying to QA
git tag dev-v1.2.1  # or dev-v20241215-1
git push origin dev-v1.2.1

# Test in isolated dev environment
# Manually destroy when done
# Then proceed with QA tag
```

#### Step 5: Production Deployment

```bash
# Create ServiceNow Change Request
# Include commit SHA and tag name in CR
# Wait for CR approval

# After CR is approved, tag for production
git checkout release/1.2.0
git tag prod-v1.2.1  # or prod-v20241215-1
git push origin prod-v1.2.1

# Workflow validates CR and deploys to production
```

#### Step 6: Post-deployment Cleanup

```bash
# Merge release branch back to main if not already done
# (Fixes should have been merged incrementally during QA cycle)
git checkout main
git merge release/1.2.0
git push origin main

# Delete release branch
git branch -d release/1.2.0
git push origin --delete release/1.2.0

# Delete QA tags
git push origin --delete qa-v1.2.0
git push origin --delete qa-v1.2.1

# Production tag and workspace remain permanently
```

### Deployment Triggers

All tag-based deployments can be configured as either automatic or manual based on team preference and requirements:

**Automatic Deployment:**
- Deploy immediately when tag is pushed
- Suitable for development and QA environments
- Faster feedback loop
- Requires confidence in quality gates

**Manual Deployment:**
- Require manual approval/trigger in GitHub Actions UI
- Common pattern for Terraform workflows:
  - Run terraform plan/validate automatically
  - Set terraform apply as manual approval step
- Common pattern for non-Terraform workflows:
  - Run build/test steps automatically
  - Set deployment step as manual approval
- Provides additional control and review opportunity
- Often used for production deployments

Teams choose deployment trigger strategy based on:
- Comfort level with automation
- Risk tolerance for each environment
- Compliance and audit requirements
- Complexity of deployment

## Hotfix Process

### When to Use Hotfixes

Create a fix branch and follow the hotfix process when:
- Critical production bug requires immediate fix
- Security vulnerability discovered in production
- Data integrity issue affecting production
- Issue cannot wait for normal release cycle
- Production is impaired or down

### Hotfix Flow

#### Step 1: Create Fix Branch from Production Tag

```bash
# Identify the current production tag
# Create fix branch from that tag
git checkout -b fix/critical-issue prod-v1.2.1
# or
git checkout -b fix/security-patch prod-v20241215-1

# Push fix branch
git push origin fix/critical-issue
```

#### Step 2: Implement Fix

```bash
# Make minimal changes to fix the issue
# Keep scope limited to the specific problem
git add .
git commit -m "fix: resolve critical production issue"
git push origin fix/critical-issue
```

#### Step 3: Test in Isolated Dev Environment

```bash
# Tag for isolated dev testing
git tag dev-v1.2.2  # or dev-v20241215-2
git push origin dev-v1.2.2

# Isolated dev environment deploys automatically
# Test the fix thoroughly
# Manually destroy when testing complete
```

**Note:** QA environment is likely being used for ongoing development, so we use isolated dev environment instead of overwriting QA.

#### Step 4: Production Deployment

```bash
# Create ServiceNow Change Request (emergency CR if needed)
# Include commit SHA and tag name in CR

# After CR approval, tag for production
git tag prod-v1.2.2  # or prod-v20241215-2
git push origin prod-v1.2.2

# Workflow validates CR and deploys immediately
```

#### Step 5: Merge Back to Main

```bash
# Create pull request from fix branch to main
# Requires 1 approval from developer who is not a committer
# Merge to ensure fix is in main branch

# Delete fix branch
git branch -d fix/critical-issue
git push origin --delete fix/critical-issue

# Delete dev tag used for testing
git push origin --delete dev-v1.2.2

# Production tag remains permanently
```

### Emergency Override

In critical situations where production is down or data loss is imminent:

1. Create fix branch and implement fix
2. Tag for production deployment
3. Deploy (ServiceNow CR can be created concurrently or post-deployment)
4. Document emergency deployment in incident report
5. Follow up with proper post-mortem review
6. Ensure retroactive CR is created for audit compliance

## Terraform Workspace Management

> **Note**: For comprehensive Infrastructure as Code standards, see [Deployment Standards](./../deployment-standards/deployment-standards.md).

### Workspace Naming Strategy

Terraform workspaces provide state isolation for different environments and deployments. Workspace names are derived from tags or branch names.

#### Official Environments (Overwrite on Each Deployment)

These environments use a static workspace name, so each deployment overwrites the previous state:

| Source | Workspace Name | Behavior |
|--------|----------------|----------|
| Main branch | `main` | Persistent workspace, state updated on each main deployment |
| QA tags (`qa-v*`) | `qa` | Persistent workspace, overwritten with each new QA tag deployment |
| Production tags (`prod-v*`) | `prod` | Persistent workspace, overwritten with each new prod tag deployment |

**Example:**
```bash
# First QA deployment
qa-v1.2.0 → workspace: qa → resources: my-app-qa-*

# QA redeployment after fix
qa-v1.2.1 → workspace: qa → resources: my-app-qa-* (same resources, updated)
```

#### Isolated Environments (Parallel Deployments)

These environments use the full scrubbed tag name as the workspace, creating completely separate state:

| Source | Workspace Name | Behavior |
|--------|----------------|----------|
| Preview tags (`preview-*`) | `preview-feature-name` | Unique workspace per preview, parallel deployments |
| Dev tags (`dev-v*`) | `dev-vX-Y-Z` | Unique workspace per dev tag, parallel deployments |

**Example:**
```bash
# Multiple parallel preview environments
preview-auth → workspace: preview-auth → resources: my-app-preview-auth-*
preview-dashboard → workspace: preview-dashboard → resources: my-app-preview-dashboard-*

# Multiple parallel dev environments
dev-v1.2.1 → workspace: dev-v1-2-1 → resources: my-app-dev-v1-2-1-*
dev-v1.2.2 → workspace: dev-v1-2-2 → resources: my-app-dev-v1-2-2-*
```

### Name Scrubbing Rules

Terraform workspace names must comply with these restrictions:
- Lowercase alphanumeric characters and hyphens only
- Maximum 90 characters
- Cannot start with `terraform-` or end with `-state`

**Scrubbing Function:**

```bash
scrub_tag_for_workspace() {
  local tag=$1
  # Convert to lowercase
  tag=$(echo "$tag" | tr '[:upper:]' '[:lower:]')
  # Replace dots (.) and slashes (/) with hyphens (-)
  tag=$(echo "$tag" | tr './' '-')
  # Remove any other special characters
  tag=$(echo "$tag" | sed 's/[^a-z0-9-]//g')
  # Truncate to 90 characters
  tag=$(echo "$tag" | cut -c1-90)
  echo "$tag"
}
```

### Tag to Workspace Mapping Examples

| Tag | Workspace Name | Notes |
|-----|----------------|-------|
| `main` (branch) | `main` | No scrubbing needed |
| `qa-v1.2.3` | `qa` | Official QA environment |
| `qa-v20241215-1` | `qa` | Official QA environment |
| `prod-v1.2.3` | `prod` | Official production environment |
| `prod-v20241215-1` | `prod` | Official production environment |
| `dev-v1.2.3` | `dev-v1-2-3` | Isolated dev (dots → hyphens) |
| `dev-v20241215-1` | `dev-v20241215-1` | Isolated dev (no changes needed) |
| `preview-my-feature` | `preview-my-feature` | Isolated preview (no changes needed) |
| `preview-User_Auth` | `preview-user-auth` | Isolated preview (lowercase, underscore removed) |

### Workspace Lifecycle

| Workspace | Lifecycle | When Deleted |
|-----------|-----------|--------------|
| `main` | Persistent | Never |
| `qa` | Persistent | Never (but state overwritten with each QA deployment) |
| `prod` | Persistent | Never |
| `preview-*` | Ephemeral | When preview environment manually destroyed |
| `dev-v*` | Ephemeral | When isolated dev environment manually destroyed |

### Workflow Implementation

**Determine Workspace from Tag/Branch:**

```yaml
- name: Determine Terraform workspace
  id: workspace
  run: |
    if [[ "$GITHUB_REF" == refs/heads/main ]]; then
      WORKSPACE="main"
    elif [[ "$GITHUB_REF" == refs/tags/qa-v* ]]; then
      WORKSPACE="qa"
    elif [[ "$GITHUB_REF" == refs/tags/prod-v* ]]; then
      WORKSPACE="prod"
    elif [[ "$GITHUB_REF" == refs/tags/dev-v* ]]; then
      TAG_NAME="${GITHUB_REF#refs/tags/}"
      WORKSPACE=$(echo "$TAG_NAME" | tr '[:upper:]' '[:lower:]' | tr './' '-' | sed 's/[^a-z0-9-]//g')
    elif [[ "$GITHUB_REF" == refs/tags/preview-* ]]; then
      TAG_NAME="${GITHUB_REF#refs/tags/}"
      WORKSPACE=$(echo "$TAG_NAME" | tr '[:upper:]' '[:lower:]' | tr './' '-' | sed 's/[^a-z0-9-]//g')
    fi
    echo "workspace=${WORKSPACE}" >> $GITHUB_OUTPUT
```

**Select or Create Workspace:**

```yaml
- name: Select or create Terraform workspace
  run: |
    terraform workspace select ${{ steps.workspace.outputs.workspace }} || \
    terraform workspace new ${{ steps.workspace.outputs.workspace }}
```

**Destroy Workspace (for ephemeral environments):**

```yaml
- name: Destroy environment and workspace
  run: |
    terraform workspace select ${{ steps.workspace.outputs.workspace }}
    terraform destroy -auto-approve
    terraform workspace select default
    terraform workspace delete ${{ steps.workspace.outputs.workspace }}
```

### State Backend Configuration

```hcl
terraform {
  backend "s3" {
    bucket         = "org-terraform-state"
    key            = "app-name/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
    
    # Workspaces automatically create separate state files:
    # s3://bucket/app-name/env:/workspace-name/terraform.tfstate
  }
}
```

### AWS Authentication for Terraform

> **Note**: For comprehensive AWS authentication and IAM role standards, see [AWS Architecture Standards](./../architecture-standards/platform-specific/aws.md).

#### Self-Hosted GitHub Actions Runners

Terraform deployments to AWS from GitHub Actions use self-hosted runners with IAM role-based authentication:

- **Runner Deployment**: Each developer grouping (team) deploys self-hosted GitHub Actions runners aligned with their GitHub repositories
- **IAM Role-Based Authentication**: Runners are configured with IAM roles appropriate for their team and use cases
- **Automatic Credential Provisioning**: IAM roles automatically provide temporary credentials via the AWS metadata service
- **Environment-Scoped Roles**: Roles are scoped by team and environment context (development, QA, production)
- **No Explicit Credentials Required**: Terraform uses the default AWS credential chain, automatically picking up credentials from the runner's IAM role

#### Terraform Provider Configuration

With self-hosted runners using IAM roles, Terraform requires no explicit provider credentials:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# No explicit credentials needed - uses default credential chain
# Credentials are automatically provided by runner's IAM role
provider "aws" {
  region = var.aws_region
  # Credentials automatically obtained from runner's IAM role
}
```

#### Security Benefits

This approach aligns with security standards:
- **Temporary Credentials**: IAM roles provide temporary credentials that rotate automatically
- **No Long-Lived Secrets**: No access keys stored in GitHub secrets or configuration files
- **Principle of Least Privilege**: Roles can be scoped precisely to team and environment needs
- **Audit Trail**: All Terraform operations are traceable to the IAM role and runner

## Resource Naming Standards

> **Note**: For AWS-specific architecture and resource standards, see [AWS Architecture Standards](./../architecture-standards/platform-specific/aws.md).

### Naming Convention

All cloud resources must include the workspace name in their identifier for traceability and environment isolation.

**Format:** `{app-name}-{workspace}-{resource-type}`

### Examples by Environment

| Workspace | Resource Type | Full Resource Name |
|-----------|--------------|-------------------|
| `main` | API Server | `my-app-main-api-server` |
| `main` | PostgreSQL Database | `my-app-main-postgres` |
| `qa` | API Server | `my-app-qa-api-server` |
| `qa` | S3 Bucket | `my-app-qa-data` |
| `prod` | API Server | `my-app-prod-api-server` |
| `prod` | Lambda Function | `my-app-prod-processor` |
| `dev-v1-2-3` | API Server | `my-app-dev-v1-2-3-api-server` |
| `dev-v1-2-3` | Database | `my-app-dev-v1-2-3-postgres` |
| `preview-auth` | API Server | `my-app-preview-auth-api-server` |
| `preview-auth` | Lambda Function | `my-app-preview-auth-processor` |

### Terraform Variable Configuration

```hcl
variable "app_name" {
  description = "Application name"
  type        = string
  default     = "my-app"
}

variable "environment" {
  description = "Environment: dev, qa, prod, preview"
  type        = string
}

variable "commit_sha" {
  description = "Git commit SHA being deployed"
  type        = string
}

# Compute resource prefix from workspace
locals {
  resource_prefix = "${var.app_name}-${terraform.workspace}"
}

# Example resource
resource "aws_instance" "api" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  tags = {
    Name        = "${local.resource_prefix}-api-server"
    Application = var.app_name
    Environment = var.environment
    Workspace   = terraform.workspace
    CommitSHA   = var.commit_sha
    ManagedBy   = "Terraform"
  }
}
```

### AWS-specific Naming Constraints

Different AWS services have different naming requirements. Resources must be named to comply with service-specific constraints.

#### S3 Buckets

**Constraints:**
- Must be lowercase
- No underscores allowed
- Must be globally unique across all AWS accounts
- Only alphanumeric characters and hyphens

**Implementation:**

```hcl
resource "aws_s3_bucket" "data" {
  # Replace underscores with hyphens, ensure lowercase
  bucket = replace(lower(local.resource_prefix), "_", "-")
  
  tags = {
    Name        = "${local.resource_prefix}-data"
    Application = var.app_name
    Environment = var.environment
    Workspace   = terraform.workspace
    CommitSHA   = var.commit_sha
  }
}
```

#### IAM Roles

**Constraints:**
- Alphanumeric characters plus: `+=,.@_-`
- Maximum 64 characters

**Implementation:**

```hcl
resource "aws_iam_role" "app_role" {
  name = "${local.resource_prefix}-app-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
  
  tags = {
    Name        = "${local.resource_prefix}-app-role"
    Application = var.app_name
    Environment = var.environment
    Workspace   = terraform.workspace
  }
}
```

#### Lambda Functions

**Constraints:**
- Alphanumeric characters, hyphens, underscores
- Maximum 64 characters

**Implementation:**

```hcl
resource "aws_lambda_function" "processor" {
  function_name = "${local.resource_prefix}-processor"
  role          = aws_iam_role.lambda_role.arn
  handler       = "index.handler"
  runtime       = "nodejs18.x"
  
  tags = {
    Name        = "${local.resource_prefix}-processor"
    Application = var.app_name
    Environment = var.environment
    Workspace   = terraform.workspace
    CommitSHA   = var.commit_sha
  }
}
```

## ServiceNow Integration

### Overview

All production deployments require an approved ServiceNow Change Request (CR). The deployment workflow validates the CR before deploying to production to ensure proper change management and audit compliance.

### Production Deployment Requirements

**Required for:**
- All deployments to production environment (`prod-v*` tags)

**Not required for:**
- Development environment deployments
- QA environment deployments
- Preview environment deployments
- Isolated dev environment deployments

### Change Request Validation

The deployment workflow validates the following before deploying to production:

**CR Must Include:**
- **Commit SHA:** The exact Git commit hash being deployed
- **Tag Name:** The production tag name (e.g., `prod-v1.2.1`)

**CR Must Pass Validation:**
- **CR Exists:** A change request is found matching the commit SHA
- **Status is Approved:** CR status must be "Approved" (not draft, pending, or rejected)
- **Valid Time Window:** Current deployment time must fall within the CR's scheduled change window

### CR Validation Timing

**Critical:** CR validation occurs immediately before the deployment step, not at workflow start.

This timing allows:
1. Tag creation to trigger the workflow
2. Terraform plan execution and review
3. Build and artifact preparation steps
4. **Then** CR validation before apply/deploy

**For Automatic Deployments:**
- CR must be in approved status from workflow start
- CR's scheduled change window must be valid when workflow begins
- Workflow will fail at deployment step if CR is not valid

**For Manual Deployments:**
- CR can be created/approved while earlier workflow steps are running
- CR must be valid when deployment step is manually triggered
- Provides flexibility for emergency changes

**Example Workflow Sequence:**

```
1. Developer creates prod tag → Workflow starts
2. Checkout code
3. Run terraform plan → Reviewed by team
4. Build artifacts
5. [VALIDATION] Check ServiceNow CR → Must pass
6. Deploy to production → Only if validation passed
7. Update ServiceNow CR with deployment status
```

### Post-deployment Updates

After successful deployment, the workflow updates the ServiceNow CR with:
- Deployment status (success/failure)
- Deployment timestamp
- Deployed commit SHA
- Environment deployed to
- Any relevant deployment notes

This provides complete audit trail and change tracking.

## Best Practices

### Branch Management

**Creating Branches:**
- Always create release branches from a clean, tested main branch
- Ensure main branch is in a known good state before branching
- Run full test suite on main before creating release branch

**Branch Lifecycle:**
- Keep release branches short-lived to minimize divergence from main
- Merge fixes from release branches back to main as soon as possible (developer may validate first)
- Delete transient branches immediately after merge or deployment
- Never keep old release branches "just in case" - use tags instead

**Tagging:**
- Never modify or delete production tags - they are permanent audit records
- Choose one tagging convention per project and document in README
- Tag naming must be consistent across the entire project
- Clean up temporary tags (preview, dev-v, qa) after use to keep repository organized

### Code Quality

> **Note**: For comprehensive code quality standards and analysis frameworks, see [Development Standards](./../development-standards/development-standards.md).

**Quality Gates:**
- All code must pass static analysis before merge
- Unit tests required with adequate coverage
- Linting must pass with no warnings
- Security scanning must show no critical vulnerabilities
- Build validation must succeed

**Terraform Planning:**
- Always review terraform plan output before applying
- Feature branches run terraform plan against main workspace
- Release branches run terraform plan against qa workspace
- Fix branches run terraform plan against prod workspace
- Never apply terraform changes without reviewing plan first

### Code Review

**Pull Request Requirements:**
- All PRs require 1 approval from a developer who is not a committer on the source branch
- Requirement applies to PRs targeting main and release branches
- Reviewers should verify:
  - Code quality and maintainability
  - Test coverage is adequate
  - Security implications are considered
  - Performance impact is understood
  - Documentation is updated if needed

**Review Process:**
- Reviews should happen within 1 business day
- Use PR templates to ensure consistent review criteria
- Resolve all conversations before merging
- Require branches to be up to date with target before merging

### Deployment Best Practices

> **Note**: For comprehensive deployment standards including rollback strategies and health checks, see [Deployment Standards](./../deployment-standards/deployment-standards.md) and [Operations Standards](./../operations-standards/operations-standards.md).

**Pre-deployment:**
- Review terraform plan output thoroughly
- Verify all tests pass in lower environments
- Ensure ServiceNow CR is approved (for production)
- Communicate deployment to relevant stakeholders
- Have rollback plan ready before deploying

**During Deployment:**
- Monitor deployment progress in real-time
- Watch application logs and metrics
- Be prepared to rollback if issues detected

**Post-deployment:**
- Run smoke tests to verify deployment success
- Monitor application health and performance
- Update change management systems
- Communicate deployment completion to stakeholders

**Manual vs Automatic:**
- Use manual approval for production deployments when appropriate
- Consider automatic deployments for dev and QA to increase velocity
- For Terraform: run plan automatically, set apply as manual approval
- For application deployments: run build/test automatically, set deploy as manual

**Deployment Strategies:**
> **Note**: For comprehensive deployment strategy standards including blue-green and canary deployments, see [Deployment Standards](./../deployment-standards/deployment-standards.md). All production deployments should use zero-downtime deployment strategies (blue-green or canary) as specified in deployment standards.

### Deployment Windows

**Recommended Windows:**
- **Development:** Anytime (automated deployments acceptable)
- **QA:** Business hours (to ensure QA team availability)
- **Production:** Tuesday-Thursday, 10am-2pm local time

**Avoid Deploying:**
- Friday afternoons (insufficient time for issue resolution before weekend)
- Monday mornings (allow time for week planning)
- End of month (financial close periods, high business activity)
- Major holidays and holiday eves
- During known high-traffic periods

### Rollback Strategy

**Quick Rollback (Emergency):**

If current production deployment fails critically:

```bash
# Option 1: Redeploy previous known-good tag
# Trigger redeployment of last working production tag
# May require workflow modification to support tag-based rollback

# Option 2: Create new production tag from previous commit
git checkout prod-v1.2.0  # Last known good version
git tag prod-v1.2.0-rollback
git push origin prod-v1.2.0-rollback
# Triggers deployment of known-good code
```

**Proper Rollback Process:**

1. Identify last known good production tag
2. Create ServiceNow CR for rollback (emergency if needed)
3. Create new production tag from last known good commit
4. Deploy via normal production process
5. Document rollback in incident report
6. Schedule post-mortem to identify root cause

**Rollback Considerations:**
- Database migrations may prevent simple rollback
- Consider data compatibility when rolling back
- Communicate rollback to all stakeholders
- Plan for potential downtime during rollback

### Testing Strategy by Branch Type

| Branch Type | Required Tests |
|-------------|----------------|
| **Feature** | Unit tests, linting, static analysis, security scans |
| **Fix** | Unit tests, linting, static analysis, security scans, regression tests |
| **Release** | Full regression suite, integration tests, performance tests, security scans |
| **Main** | Unit tests, integration tests, security scans |

**Additional Testing:**
- Smoke tests after every deployment
- Performance tests before production
- Security penetration testing periodically
- Disaster recovery testing quarterly

## Decision Trees

### When to Create a Release Branch

**Always create a release branch when promoting to QA.**

This is standard practice, not optional. The release branch:
- Provides isolation for applying QA fixes
- Allows ongoing development to continue on main
- Can be used to redeploy to dev for testing fixes
- Serves as staging area for production deployment
- Is deleted after successful production deployment

### Which Tagging Convention to Choose

**Use Semantic Versioning (Convention 1) if:**
- Building libraries or APIs that other teams/projects consume
- Need to communicate compatibility and breaking changes clearly
- External consumers depend on your versioning scheme
- Project has defined compatibility contracts
- Following industry standards for your technology (npm, Maven, etc.)

**Use Date-based Versioning (Convention 2) if:**
- Building internal applications
- Following sprint-based or time-boxed release cycles
- Release delivery date is more important than semantic meaning
- Practicing continuous delivery
- Version numbers don't communicate compatibility contracts

**Once chosen:**
- Document the convention in project README
- Remain consistent throughout project lifecycle
- Enforce convention through CI checks if needed

### Which Branch Type to Create

**Create `feature/*` branch when:**
- Developing new features
- Fixing bugs in development or QA environments
- Making any changes to non-production code
- Starting any new development work
- Can be created from `main` for general development or from `release/*` for release-specific fixes

**Create `fix/*` branch when:**
- Fixing critical production bugs (hotfixes only)
- Patching security vulnerabilities in production
- Emergency production fixes that can't wait for normal release cycle
- **Remember:** Fix branches plan against prod workspace

**Create `release/*` branch when:**
- Promoting code to QA (always created, not optional)
- Need isolation from ongoing main branch development
- Starting QA cycle for a release

### When to Use Which Tag

**Use `preview-<feature-name>` tag when:**
- Need to test feature in isolation before merging to main
- Want to demo feature to stakeholders
- Need separate environment for feature testing
- Remember to manually destroy when done

**Use `dev-vX.Y.Z` tag when:**
- Testing fixes from release branch in dev before QA
- Testing hotfixes in dev before production
- Need isolated dev environment parallel to official dev
- Remember to manually destroy when done

**Use `qa-vX.Y.Z` tag when:**
- Deploying to QA for quality assurance testing
- Ready for user acceptance testing
- Created from release branch
- Delete after successful production deployment

**Use `prod-vX.Y.Z` tag when:**
- Deploying to production
- ServiceNow CR is approved
- QA has approved the release
- **Never delete** - permanent audit record

## Conclusion

These Git workflow standards ensure reliable, traceable, and maintainable development and deployment processes. The tag-driven deployment model provides clear audit trails while supporting both simple continuous deployment scenarios and complex multi-environment release cycles with change management integration.

Regular review and updates ensure these standards remain relevant as development workflows and deployment technologies evolve. Teams should adapt these standards to their specific needs while maintaining the core principles of tag-driven deployments, branch lifecycle management, and infrastructure state isolation.

