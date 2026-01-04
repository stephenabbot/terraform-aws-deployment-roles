# Prerequisites

## Foundation Infrastructure

This project requires the [foundation-terraform-bootstrap](https://github.com/stephenabbot/foundation-terraform-bootstrap) to be deployed first. The foundation provides:

- S3 bucket for Terraform state storage with versioning and encryption
- DynamoDB table for state locking to prevent concurrent modifications
- GitHub OIDC provider for secure authentication without long-lived credentials
- Foundation deployment role for initial infrastructure management

## Required Tools

### Local Development Environment

- **OpenTofu 1.0+** - Infrastructure as Code tool (Terraform-compatible)
- **AWS CLI 2.x** - AWS service interaction and credential management
- **Bash 4.x+** - Shell scripting for automation scripts
- **jq** - Command-line JSON processor for script operations
- **Git** - Version control and repository metadata extraction
- **GitHub CLI (gh)** - Repository variable management during bootstrap

### Installation Commands

```bash
# macOS with Homebrew
brew install opentofu awscli jq gh

# Ubuntu/Debian
sudo apt update
sudo apt install -y jq git
# Install OpenTofu, AWS CLI, and GitHub CLI from official sources

# Verify installations
tofu version
aws --version
jq --version
gh --version
```

## Git Repository Requirements

### Repository Configuration

- Initialized git repository with GitHub remote origin
- Clean working directory with no uncommitted changes
- Remote origin URL must match target repository for role scoping
- Repository must be accessible to GitHub Actions workflows

### Repository Structure Validation

```bash
# Verify git repository
git remote get-url origin
git status --porcelain

# Should show GitHub URL and no uncommitted changes
```

## AWS Permissions

### Bootstrap Permissions (One-time Setup)

The principal running bootstrap requires:

- **GitHub Repository Admin** - Set repository variables via GitHub CLI
- **AWS Account Access** - Read account ID for role ARN construction

### Deployment Permissions

The deployment principal needs comprehensive IAM permissions:

#### IAM Permissions
- `iam:CreateRole` - Create deployment roles
- `iam:CreatePolicy` - Create custom policies from JSON files
- `iam:AttachRolePolicy` - Attach policies to roles
- `iam:GetRole`, `iam:GetPolicy` - Read existing resources
- `iam:ListRoles`, `iam:ListPolicies` - List resources for validation
- `iam:DeleteRole`, `iam:DeletePolicy` - Cleanup during destroy operations
- `iam:DetachRolePolicy` - Remove policy attachments
- `iam:ListRoleTags`, `iam:TagRole` - Resource tagging operations

#### SSM Parameter Store Permissions
- `ssm:PutParameter` - Store role ARNs for service discovery
- `ssm:GetParameter` - Read foundation configuration and role ARNs
- `ssm:DeleteParameter` - Remove parameters during cleanup
- `ssm:GetParameters`, `ssm:GetParametersByPath` - Batch parameter operations

#### Backend Permissions
- `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject` - State file operations
- `s3:ListBucket` - Bucket access validation
- `dynamodb:GetItem`, `dynamodb:PutItem`, `dynamodb:DeleteItem` - State locking
- `dynamodb:Scan` - Lock table operations

#### Additional Permissions
- `sts:AssumeRole` - Assume foundation deployment role (if using)
- `sts:GetCallerIdentity` - Account validation and tagging

## Bootstrap Process

### Purpose and Requirements

Bootstrap configures the `AWS_ACCOUNT_ID` GitHub repository variable that workflows need to construct deployment role ARNs. This is a one-time setup requiring repository admin permissions.

### Bootstrap Execution

```bash
# Clone repository
git clone https://github.com/stephenabbot/foundation-iam-deploy-roles.git
cd foundation-iam-deploy-roles

# Authenticate with GitHub CLI
gh auth login

# Run bootstrap (requires admin permissions)
./scripts/bootstrap.sh

# Verify setup
./scripts/verify-prerequisites.sh
```

### What Bootstrap Does

1. **Account Detection** - Retrieves AWS account ID from current credentials
2. **Variable Management** - Sets or updates `AWS_ACCOUNT_ID` GitHub repository variable
3. **Validation** - Confirms configuration is accessible to workflows
4. **Idempotency** - Safe to run multiple times, updates only when needed

## Validation Commands

### Verify Prerequisites

```bash
# Run comprehensive validation
./scripts/verify-prerequisites.sh

# Manual validation steps
aws sts get-caller-identity
git remote get-url origin
gh variable list | grep AWS_ACCOUNT_ID
```

### Check Foundation Dependencies

```bash
# Verify foundation SSM parameters exist
aws ssm get-parameter --name "/terraform/foundation/s3-state-bucket"
aws ssm get-parameter --name "/terraform/foundation/dynamodb-lock-table"
aws ssm get-parameter --name "/terraform/foundation/oidc-provider"
```

### Test OIDC Provider

```bash
# Verify OIDC provider configuration
OIDC_ARN=$(aws ssm get-parameter --name "/terraform/foundation/oidc-provider" --query Parameter.Value --output text)
aws iam get-open-id-connect-provider --open-id-connect-provider-arn "$OIDC_ARN"
```

## Troubleshooting Prerequisites

### GitHub CLI Authentication

**Problem**: `gh auth status` fails
**Solution**:
```bash
gh auth login
# Follow interactive prompts for authentication
```

### Missing Admin Permissions

**Problem**: Bootstrap fails with permission errors
**Solutions**:
- Contact repository owner to run bootstrap
- Request admin permissions for your GitHub account
- Use organization-level GitHub App with appropriate permissions

### AWS Account Mismatch

**Problem**: Bootstrap detects wrong AWS account
**Solution**:
```bash
# Verify current credentials
aws sts get-caller-identity

# Switch to correct profile if needed
export AWS_PROFILE=correct-profile
aws sts get-caller-identity
```

### Foundation Not Deployed

**Problem**: SSM parameters not found
**Solution**:
1. Deploy [foundation-terraform-bootstrap](https://github.com/stephenabbot/foundation-terraform-bootstrap) first
2. Verify foundation deployment completed successfully
3. Check foundation outputs for SSM parameter creation

### Tool Version Issues

**Problem**: Incompatible tool versions
**Solutions**:
```bash
# Update OpenTofu
brew upgrade opentofu

# Update AWS CLI
pip install --upgrade awscli

# Update GitHub CLI
brew upgrade gh
```

## Security Considerations

### Credential Management

- Never commit AWS credentials to repository
- Use AWS CLI profiles or environment variables for local development
- GitHub Actions uses OIDC federation, no stored credentials required
- Bootstrap requires temporary admin access for variable setup only

### Permission Boundaries

- Foundation deployment role has broad permissions for infrastructure setup
- Project-specific roles should follow least privilege principle
- Regular audit of deployed roles and their permissions
- Use CloudTrail for comprehensive access logging

### Network Security

- All API calls use HTTPS/TLS encryption
- No network connectivity requirements beyond standard AWS API access
- GitHub Actions runs in GitHub-managed infrastructure
- No custom networking or VPC requirements
