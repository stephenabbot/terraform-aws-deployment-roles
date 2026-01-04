# Architecture

## Infrastructure Ecosystem

This project operates within a layered infrastructure ecosystem designed for secure, scalable AWS deployments:

### Layer 1: Foundation Bootstrap
**[foundation-terraform-bootstrap](https://github.com/stephenabbot/foundation-terraform-bootstrap)** - Bootstrap Terraform backend infrastructure using CloudFormation
- S3 bucket for state storage with versioning and encryption
- DynamoDB table for state locking and concurrent access protection
- GitHub OIDC provider for secure authentication without long-lived credentials
- Foundation deployment role with broad permissions for infrastructure setup

### Layer 2: Deployment Roles
**foundation-iam-deploy-roles** (this project) - Create IAM roles for GitHub Actions deployment workflows
- Repository-scoped IAM roles with OIDC trust policies
- Custom deployment policies from JSON configuration files
- SSM Parameter Store integration for service discovery
- Automated role lifecycle management

### Layer 3: Application Infrastructure
**[website-infrastructure](https://github.com/stephenabbot/website-infrastructure)** - Application-specific AWS resources
- CloudFront distributions and S3 buckets for static hosting
- Route 53 DNS configuration and SSL certificates
- Application-specific IAM policies and resources

### Layer 4: Application Content
**[website_denverbytes_com](https://github.com/stephenabbot/website_denverbytes_com)** - Static website content and build process
- Static site generation and content management
- Deployment workflows using roles from Layer 2
- Content delivery through infrastructure from Layer 3

## Authentication Flow

The OIDC authentication flow eliminates the need for long-lived AWS credentials:

```
GitHub Actions → OIDC Token → AWS STS → Temporary Credentials → AWS Resources
```

### Detailed Authentication Process

1. **Token Request**: GitHub Actions workflow requests OIDC token from GitHub
2. **Token Validation**: AWS OIDC provider validates token signature and claims
3. **Trust Policy Evaluation**: IAM role trust policy validates repository and workflow context
4. **Credential Issuance**: AWS STS issues temporary credentials (15 minutes - 12 hours)
5. **Resource Access**: Temporary credentials used for AWS API calls within workflow

### Security Boundaries

- **Repository Isolation**: Each role scoped to single GitHub repository
- **Environment Separation**: Separate roles per environment (prd, dev, staging)
- **Temporal Credentials**: No long-lived access keys stored anywhere
- **Audit Trail**: All API calls logged through CloudTrail with repository context

## Project Structure

The project uses dynamic discovery to automatically detect and deploy roles for all configured projects:

```
projects/
├── project-name/
│   └── environment/
│       ├── terraform.tfvars          # Environment-specific variables
│       └── policies/
│           └── deployment-policy.json # IAM permissions for this project/environment
└── another-project/
    └── prd/
        └── policies/
            └── deployment-policy.json
```

### Dynamic Discovery Process

1. **File System Scan**: OpenTofu scans `projects/*/*/policies/deployment-policy.json`
2. **Path Parsing**: Extracts project name and environment from directory structure
3. **Role Generation**: Creates IAM role and policy for each discovered combination
4. **SSM Publishing**: Stores role ARNs in Parameter Store for service discovery

## Resource Naming Conventions

Consistent naming enables automated discovery and management:

### IAM Resources
- **Roles**: `gharole-{project}-{environment}` (GitHub Actions Role)
- **Policies**: `ghpolicy-{project}-{environment}` (GitHub Policy)

### SSM Parameters
- **Role ARNs**: `/deployment-roles/{project}/role-arn`
- **Foundation Config**: `/terraform/foundation/{resource-type}`

### Backend Resources
- **State Keys**: `deployment-roles/{org-repo}/terraform.tfstate`
- **Lock Keys**: `deployment-roles/{org-repo}/terraform.tfstate-md5`

## Trust Policy Configuration

Trust policies use OIDC claims to restrict role assumption to specific repositories and contexts:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:org/project:*"
        }
      }
    }
  ]
}
```

### Trust Policy Claims

- **aud (Audience)**: Must be "sts.amazonaws.com" for AWS STS
- **sub (Subject)**: Repository path with wildcard for branches/environments
- **iss (Issuer)**: GitHub OIDC provider URL for token validation

## Backend Architecture

The backend provides centralized state management and locking for all deployment roles:

### S3 State Storage
- **Bucket**: Discovered from `/terraform/foundation/s3-state-bucket` SSM parameter
- **Encryption**: AES-256 server-side encryption enabled
- **Versioning**: Full version history for state recovery
- **Access Control**: Restricted to foundation and deployment roles

### DynamoDB State Locking
- **Table**: Discovered from `/terraform/foundation/dynamodb-lock-table` SSM parameter
- **Lock Key**: Combination of state bucket and key for uniqueness
- **TTL**: Automatic cleanup of stale locks after timeout
- **Consistency**: Strong consistency for concurrent access protection

### State Key Strategy
- **Pattern**: `deployment-roles/{org-repo}/terraform.tfstate`
- **Isolation**: Each repository has separate state file
- **Discovery**: Automatic construction from GitHub repository context

## Service Discovery

Role ARNs are published to SSM Parameter Store for consuming projects to discover their deployment roles:

### Parameter Structure
```
/deployment-roles/
├── project-1/
│   └── role-arn → arn:aws:iam::ACCOUNT:role/gharole-project-1-prd
├── project-2/
│   └── role-arn → arn:aws:iam::ACCOUNT:role/gharole-project-2-prd
└── project-3/
    └── role-arn → arn:aws:iam::ACCOUNT:role/gharole-project-3-prd
```

### Discovery Process
Consuming projects discover their deployment roles using AWS CLI or SDK:

```bash
# Bash example
ROLE_ARN=$(aws ssm get-parameter --name "/deployment-roles/my-project/role-arn" --query Parameter.Value --output text)

# GitHub Actions example
- name: Get deployment role
  id: role
  run: |
    ROLE_ARN=$(aws ssm get-parameter --name "/deployment-roles/${{ github.event.repository.name }}/role-arn" --query Parameter.Value --output text)
    echo "role-arn=$ROLE_ARN" >> $GITHUB_OUTPUT
```

## GitHub Actions Integration

Workflows use the OIDC provider and role ARNs for secure AWS access:

### Workflow Configuration
```yaml
permissions:
  id-token: write  # Required for OIDC token request
  contents: read   # Required for repository checkout

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ env.ROLE_ARN }}
          aws-region: us-east-1
          
      - name: Deploy Infrastructure
        run: |
          # AWS CLI commands use temporary credentials automatically
          aws s3 ls
```

### Credential Lifecycle
1. **Token Request**: Workflow requests OIDC token with repository claims
2. **Role Assumption**: `aws-actions/configure-aws-credentials` assumes deployment role
3. **Environment Setup**: Action sets AWS environment variables for subsequent steps
4. **Automatic Refresh**: Action handles credential refresh if workflow exceeds token lifetime
5. **Cleanup**: Credentials automatically expire when workflow completes

## Operational Patterns

### Deployment Workflow

The deployment process follows a consistent pattern across all projects:

1. **Prerequisite Validation**
   - Verify git repository is clean and pushed
   - Check AWS credentials and account access
   - Validate foundation infrastructure exists

2. **Foundation Role Assumption** (Optional)
   - Assume foundation deployment role if using layered permissions
   - Alternative: Use direct AWS credentials for deployment

3. **Backend Configuration Discovery**
   - Retrieve S3 bucket and DynamoDB table from SSM parameters
   - Construct state key from repository information
   - Initialize OpenTofu with dynamic backend configuration

4. **Resource Deployment**
   - Plan infrastructure changes with OpenTofu
   - Apply changes with automatic approval in CI/CD
   - Validate deployment success and resource creation

5. **Verification and Cleanup**
   - Verify role creation and policy attachment
   - Test role assumption and basic permissions
   - Clean up temporary files and credentials

### Self-Deployment Prevention

The system prevents circular dependencies through git repository detection:

```bash
# Git repository detection
REMOTE_URL=$(git remote get-url origin)
if [[ "$REMOTE_URL" =~ github\.com[:/]([^/]+)/([^/.]+) ]]; then
  CURRENT_PROJECT="${BASH_REMATCH[2]}"
  if [ "$PROJECT_NAME" = "$CURRENT_PROJECT" ]; then
    echo "Error: Cannot create deployment role for this project itself"
    exit 1
  fi
fi
```

This prevents the foundation-iam-deploy-roles project from creating a deployment role for itself, which would create a circular dependency.

### Multi-Project Support

The architecture supports multiple projects through:

- **Dynamic Discovery**: Automatic detection of all project configurations
- **Isolated State**: Separate OpenTofu state per repository
- **Independent Lifecycles**: Projects can be created, updated, and destroyed independently
- **Resource Isolation**: Each project has dedicated IAM resources with no cross-project access

## Security Model

### Defense in Depth

The architecture implements multiple security layers:

1. **Authentication Layer**: OIDC federation eliminates stored credentials
2. **Authorization Layer**: IAM roles and policies enforce least privilege
3. **Network Layer**: All communication over HTTPS/TLS
4. **Audit Layer**: CloudTrail logging for all API calls
5. **Temporal Layer**: Short-lived credentials with automatic expiration

### Threat Mitigation

- **Credential Theft**: No long-lived credentials to steal
- **Privilege Escalation**: Repository-scoped trust policies prevent cross-project access
- **Insider Threats**: Audit trail links all actions to specific repositories and workflows
- **Supply Chain**: OIDC tokens include repository and workflow context for validation
- **Lateral Movement**: Each project isolated with dedicated IAM resources

### Compliance Considerations

- **Audit Requirements**: CloudTrail provides comprehensive API logging
- **Access Control**: Role-based access with repository-level granularity
- **Data Protection**: State files encrypted at rest and in transit
- **Change Management**: Infrastructure as Code with version control and approval workflows
