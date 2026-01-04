# GitHub Actions Integration

## What Problem This Solves

Manual deployment of IAM roles from local development environments creates operational overhead and prevents automated CI/CD workflows. Teams need secure, automated deployment of infrastructure changes without managing long-lived credentials or manual deployment steps.

- Manual deployment processes don't scale across teams and repositories
- Local-only deployment prevents automated testing and validation in CI/CD pipelines
- Managing AWS credentials for CI/CD creates security risks and operational complexity
- Teams need consistent deployment processes between local development and automated workflows

## What This Solution Does

Enables automated deployment through GitHub Actions using OIDC authentication while maintaining local-first development workflow. The solution automatically manages GitHub repository variables and uses the same deployment scripts in both environments.

- Provides secure GitHub Actions workflows using OIDC authentication without long-lived credentials
- Automatically manages AWS account ID configuration through local-first deployment pattern
- Uses identical deployment scripts locally and in CI/CD for consistency
- Eliminates manual GitHub repository configuration through automated variable management

## Deployment Flow

### Phase 1: Local First Deployment

```text
Developer    deploy.sh    verify-prereq.sh    AWS STS    GitHub API    Infrastructure
    |           |              |              |           |               |
    |--run----->|              |              |           |               |
    |           |--call------->|              |           |               |
    |           |              |--check gh--->|           |               |
    |           |              |--check aws-->|           |               |
    |           |<--pass-------|              |           |               |
    |           |--get-account--------------->|           |               |
    |           |<--account-id----------------|           |               |
    |           |--get/set var--------------------------->|               |
    |           |<--var updated---------------------------|               |
    |           |--assume role--------------->|           |               |
    |           |--deploy-------------------------------->|               |
    |           |<--success-------------------------------|               |
    |<--done----|              |              |           |               |
```

### Phase 2: GitHub Actions Deployment

```text
GH Workflow  deploy.sh    verify-prereq.sh     OIDC       Foundation     Infrastructure
    |             |              |               |           Role            |
    |--trigger--->|              |               |           |               |
    |             |--call------->|               |           |               |
    |             |              |--check tools->|           |               |
    |             |<--pass-------|               |           |               |
    |--read var-->|              |               |           |               |
    |--use OIDC----------------->|               |           |               |
    |<--assume-------------------|-------------->|           |               |
    |             |--deploy with assumed role--------------->|               |
    |             |<--success--------------------------------|               |
    |<--done------|              |               |           |               |
```

## How It Works

### Local Development Workflow

1. Developer runs `./scripts/deploy.sh` with local AWS credentials
2. Script automatically detects current AWS account ID from session
3. Script manages GitHub repository variable `AWS_ACCOUNT_ID` (create/update/verify)
4. Infrastructure deploys using local credentials
5. GitHub Actions capability is now configured automatically

### GitHub Actions Workflow

1. Workflow reads `vars.AWS_ACCOUNT_ID` to construct foundation deployment role ARN
2. Uses OIDC to assume the foundation deployment role (trust relationship pre-configured)
3. Runs identical deployment scripts with assumed role credentials
4. Infrastructure deploys through automated CI/CD pipeline

## Key Benefits

### Operational Excellence

- **Local-First Pattern**: Always works locally before automation, ensuring reliable workflows
- **Self-Configuring**: Automatically manages GitHub repository variables without manual setup
- **Consistent Scripts**: Identical deployment logic locally and in CI/CD eliminates environment drift
- **Self-Healing**: Automatically detects and corrects AWS account ID mismatches

### Security

- **OIDC Authentication**: No long-lived credentials stored in GitHub secrets
- **Least Privilege**: Uses foundation deployment role with scoped permissions
- **Account Verification**: Ensures GitHub variables match actual AWS session
- **Automated Trust**: Leverages pre-configured OIDC trust relationships

### Scalability

- **Zero Manual Setup**: No per-repository GitHub configuration required
- **Account Agnostic**: Automatically adapts to different AWS accounts
- **Team Ready**: Works consistently across development teams
- **Multi-Repository**: Scales to any number of repositories using this pattern

## Prerequisites

The GitHub Actions integration requires:

- Foundation infrastructure deployed via [foundation-terraform-bootstrap](https://github.com/stephenabbot/foundation-terraform-bootstrap)
- GitHub CLI installed locally (automatically verified by prerequisite checks)
- Local AWS credentials for initial deployment
- Repository configured with OIDC trust in foundation deployment role

## Workflow Triggers

GitHub Actions workflows trigger on:

- **Push to main branch**: Automatic deployment of infrastructure changes
- **Pull requests**: Validation and planning without deployment
- **Manual dispatch**: On-demand deployment through GitHub UI

This approach ensures infrastructure changes are validated in pull requests and automatically deployed when merged to main branch.
