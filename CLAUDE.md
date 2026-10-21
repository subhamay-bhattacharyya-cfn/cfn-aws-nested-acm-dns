# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a **CloudFormation template repository** that provisions an AWS Certificate Manager (ACM) public certificate validated through DNS in an existing Route 53 hosted zone, plus a Route 53 record that points a domain at a target (ALB, CloudFront, or any hostname).

**Key characteristics:**

- Single root template at `cloudformation/template.yaml` (no nested stacks)
- DNS validation of the certificate through Route 53 (`DomainValidationOptions`)
- Certificate Transparency logging enabled
- Optional Subject Alternative Names (SANs)
- `CNAME` or `A` record for the domain
- Automated semantic versioning and releases
- AWS OIDC authentication for CI/CD deployments

## Project Structure

```text
cloudformation/
├── template.yaml                  # ACM certificate + Route 53 DNS record
├── parameters.json                # Parameter values for the stack
└── stack-config.json              # Stack name, template path, and parameter file used by CI

.github/workflows/
├── ci.yaml                        # Loads stack config, calls the reusable CI build workflow, changelog and release on main
├── release.yaml                   # Release workflow
├── create-branch.yaml             # Auto-create feature branches from issues
└── (claude, notify, setup-environments, claude-code-review workflows)

scripts/plugins/
├── release.config.js              # Semantic-release configuration
└── (other release plugins)        # Custom commit analysis, notes generation

.claude/
├── settings.json                  # Claude Code workspace settings
└── settings.local.json            # Local overrides

.devcontainer/
└── devcontainer.json              # Dev container setup (Node.js 20)

package.json                       # Dependencies: semantic-release, commitizen
README.md                          # Template documentation and usage examples
```

## Development Commands

### Install dependencies

```bash
npm ci
```

### Trigger semantic release (usually automatic on main)

```bash
npm run release
```

### Commit with conventional commit format

```bash
npx cz commit
```

Select `feat`, `fix`, or `chore` type. Only `feat` and `fix` trigger releases.

## Key Architecture Concepts

### Resources

`cloudformation/template.yaml` creates:

- **`Certificate`** (`AWS::CertificateManager::Certificate`): public certificate for `DomainName`, validated via DNS in `HostedZoneId`. Its `Name` tag is `{ProjectName}-{DomainName}-{Environment}`, with `-{CiSuffix}` appended when `CiSuffix` is set
- **`DnsRecord`** (`AWS::Route53::RecordSet`): `CNAME` or `A` record for `DomainName` pointing at `DnsTargetValue`

### Conditions

- `HasSubjectAlternativeNames`: true when `SubjectAlternativeNames` is non-empty
- `IsCnameRecord`: true when `RecordType` is `CNAME` (TTL is applied only for CNAME)
- `HasCiSuffix`: true when `CiSuffix` is non-empty (appended to the certificate `Name` tag)

### Outputs

- `CertificateArn`: exported as `<stack-name>-CertificateArn` for use by ALB listeners, CloudFront, API Gateway, etc.
- `DomainName`: the domain secured by the certificate
- `DnsRecordName`: the record created in the hosted zone

## Key Files to Understand

### `cloudformation/template.yaml`

**Key inputs:**

- `ProjectName` (required): project prefix for the certificate `Name` tag
- `Environment` (required): environment label for the `Name` tag (e.g. `devl`, `stag`, `prod`)
- `CiSuffix` (default `""`): optional suffix for CI/CD unique deployments, used in the `Name` tag
- `HostedZoneId`: existing public Route 53 hosted zone ID
- `DomainName` (required): FQDN, a leading `*.` is allowed for wildcard certificates
- `SubjectAlternativeNames` (optional): comma-separated extra names
- `RecordType` (default `CNAME`): `CNAME` or `A`
- `DnsTargetValue` (required): hostname for CNAME, IPv4 address for A
- `DnsTTL` (default `300`): TTL in seconds, CNAME only
- `ValidationMethod` (default `DNS`): only `DNS` is supported

**Parameter groups:** The `Metadata` section (`AWS::CloudFormation::Interface`) groups parameters in the console into Project, Certificate, and DNS record.

**Known limitation:** `DomainValidationOptions` lists only `DomainName`. Validation records are not created for SAN entries, so a stack with SANs will wait on those names until they are validated. Fixing this requires one `DomainValidationOptions` entry per SAN or a custom resource, since CloudFormation has no loop without a transform.

### `cloudformation/stack-config.json`

Read by CI to determine:

- `stackName`: CloudFormation stack name
- `templateFile`: `cloudformation/template.yaml`
- `parameterFile`: `cloudformation/parameters.json`

### `cloudformation/parameters.json`

Parameter values for the deployed stack. Update this file when parameters are added or changed in the template.

### `.github/workflows/ci.yaml`

**Triggered on:**

- Manual `workflow_dispatch`
- Pushes to `main`, `feature/**`, and `bug/**` branches, when `cloudformation/**` or the workflow file changes
- Pull requests targeting `main`, when `cloudformation/**` changes

**Process:**

1. **load-config:** reads `cloudformation/stack-config.json` and exposes the stack name, template, and parameter file as outputs
2. **ci-build:** calls the reusable workflow `subhamay-bhattacharyya-gha/cfn-ci-build-reusable-wf` with the `ci` environment, stack name, template file, and parameter file
3. **generate-changelog** (main only): runs git-cliff and commits `CHANGELOG.md`
4. **create-release** (main only): creates the GitHub release

**Permissions:** `actions: write`, `contents: write`, `id-token: write` (OIDC).

### `.github/workflows/release.yaml`

**Triggered:** On push to main

**Release rules:**

- `feat:` → MINOR bump (0.1.0 → 0.2.0)
- `fix:` → PATCH bump (0.1.0 → 0.1.1)
- `BREAKING CHANGE:` → MAJOR bump (0.1.0 → 1.0.0)
- Other commits → no release

## Testing & Validation

**Validate the template:**

```bash
aws cloudformation validate-template --template-body file://cloudformation/template.yaml
```

**Manual stack deployment:**

```bash
aws cloudformation deploy \
  --template-file cloudformation/template.yaml \
  --stack-name acm-app-dev \
  --parameter-overrides \
    ProjectName=cfn-nstd \
    Environment=devl \
    HostedZoneId=Z0123456789ABCDEF \
    DomainName=app.example.com \
    DnsTargetValue=my-alb-123.us-east-1.elb.amazonaws.com \
  --region us-east-1
```

CloudFront requires certificates in `us-east-1`.

## AWS Credentials & Environment Variables

**GitHub environment `ci` must be configured with:**

- `AWS_REGION`: CloudFormation deployment region
- `AWS_ACCOUNT_ID`: AWS account to deploy into
- `OIDC_ROLE_NAME`: IAM role name for OIDC trust (uses `arn:aws:iam::{ACCOUNT_ID}:role/{ROLE_NAME}`)

**OIDC setup:** The CI workflow uses AWS OIDC for keyless auth. The GitHub OIDC provider must trust the specified role.

## Conventional Commits & Release Flow

This repo enforces conventional commits to drive semantic versioning:

```bash
npx cz commit
```

Commit types:

- `feat: add support for X` → triggers MINOR release
- `fix: correct behavior of Y` → triggers PATCH release
- `chore: update deps` → no release
- `docs: clarify README` → no release

Only commits to `main` trigger releases. Feature branches use this format but releases happen on merge to main.

## When Modifying the Template

1. **Edit `cloudformation/template.yaml`**
2. **Update `cloudformation/parameters.json`** if parameters are added or changed, and keep the `Metadata` parameter groups in sync
3. **Test locally** with `aws cloudformation validate-template`
4. **Create a PR** with a conventional commit message (e.g., `feat: add CloudFront-friendly record type`)
5. **CI validates and deploys** through the reusable workflow
6. **Merge to main** → changelog and GitHub release are created

## Dev Container

Pre-configured with:

- Node.js 20
- GitHub Copilot extension

Use via VS Code: `code --remote-container-url <repo-url>`

## Current Branch

Main branch is the release branch. Feature work branches from here and merges back via PR. Branch naming follows: `{type}/CFN-{issue-number}-{slug}` (e.g., `feature/CFN-42-add-encryption`).
