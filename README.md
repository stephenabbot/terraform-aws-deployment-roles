# Foundation IAM Deploy Roles

## GitHub Actions OIDC Authentication Infrastructure

This project creates IAM roles for GitHub Actions OIDC authentication as part of a layered infrastructure ecosystem. The [foundation-terraform-bootstrap](https://github.com/stephenabbot/foundation-terraform-bootstrap) provides the foundation infrastructure, this project creates deployment roles, and consuming projects like [website-infrastructure](https://github.com/stephenabbot/website-infrastructure) use the roles for secure AWS deployments without long-lived credentials.

## What Problem This Project Solves

GitHub Actions workflows require AWS credentials to deploy infrastructure and manage resources, creating several security and operational challenges:

- Long-lived AWS access keys stored as GitHub secrets create security risks and credential management overhead
- Manual credential rotation processes are error-prone and difficult to scale across multiple repositories
- Broad IAM permissions across projects violate the principle of least privilege
- No audit trail linking specific deployments to GitHub repository contexts
- Credential sharing between projects creates blast radius concerns for security incidents

## What This Project Does

This project implements secure, temporary credential authentication for GitHub Actions using OIDC federation:

- Creates repository-scoped IAM roles with GitHub OIDC trust policies for secure authentication
- Generates deployment policies from JSON configuration files for granular permission management
- Publishes role ARNs to SSM Parameter Store enabling service discovery for consuming projects
- Enables GitHub Actions to assume AWS roles using temporary credentials without stored secrets
- Provides automated role lifecycle management through infrastructure as code
- Implements consistent resource tagging for cost allocation and operational visibility

## What This Project Changes

This project creates and manages AWS resources that enable secure GitHub Actions authentication:

### Resources Created/Managed

- **IAM Roles**: `gharole-{project}-{environment}` with OIDC trust policies scoped to specific GitHub repositories
- **IAM Policies**: `ghpolicy-{project}-{environment}` with custom permissions from JSON configuration files
- **SSM Parameters**: `/deployment-roles/{project-name}/role-arn` for service discovery by consuming projects
- **Resource Tags**: Consistent tagging across all resources for cost allocation and operational management

### Functional Changes

- GitHub Actions workflows can authenticate to AWS using temporary credentials via OIDC federation
- Consuming projects discover their deployment role ARNs through SSM Parameter Store lookups
- Repository-scoped trust policies prevent cross-repository role assumption for enhanced security
- Automated backend configuration discovery eliminates manual state management setup

## Quick Start

### Local Development
```bash
# One-time setup (requires admin permissions)
./scripts/bootstrap.sh
./scripts/verify-prerequisites.sh

# Create and deploy project
./scripts/create-project.sh my-project
# Edit projects/my-project/prd/policies/deployment-policy.json
./scripts/deploy.sh
```

### GitHub Actions (Automated)
The project includes three GitHub Actions workflows for automated deployment and validation. See [docs/prerequisites.md](docs/prerequisites.md) for setup requirements and [docs/troubleshooting.md](docs/troubleshooting.md) for common issues.

- **Deploy** - Automatically deploys on push to main branch
- **Destroy** - Manual workflow to destroy all resources (requires confirmation)
- **Validate Foundation** - Tests OIDC authentication and foundation access

## AWS Well-Architected Framework

This project demonstrates alignment with AWS Well-Architected Framework principles:

### Security
- **OIDC Federation**: Eliminates long-lived credentials through temporary token-based authentication
- **Repository Scoping**: Trust policies restrict role assumption to specific GitHub repositories
- **Least Privilege**: Custom policies enable granular permission management per project
- **Audit Trail**: CloudTrail integration provides comprehensive access logging and attribution

### Operational Excellence
- **Infrastructure as Code**: OpenTofu manages all resources with version control and change tracking
- **Automated Deployment**: GitHub Actions workflows provide consistent, repeatable deployment processes
- **Service Discovery**: SSM Parameter Store enables automated role ARN discovery for consuming projects
- **Resource Tagging**: Consistent tagging strategy supports operational visibility and cost allocation

### Reliability
- **State Management**: S3 backend with DynamoDB locking ensures consistent infrastructure state
- **Validation Workflows**: Automated testing verifies OIDC authentication and foundation integration
- **Error Recovery**: Deployment scripts include prerequisite validation and error handling

### Cost Optimization
- **Serverless Architecture**: No compute resources or ongoing operational costs
- **Resource Tagging**: Comprehensive tagging enables detailed cost allocation and analysis
- **Efficient Resource Usage**: Minimal IAM resources with no data transfer or storage costs

## Technologies Used

| Technology | Purpose | Implementation |
|------------|---------|----------------|
| Kiro CLI with Claude | Primary development tool | AI-assisted infrastructure development, documentation generation, and code optimization |
| OpenTofu | Infrastructure as Code | Terraform-compatible tool for AWS resource provisioning and state management |
| AWS IAM | Identity and Access Management | Role creation, policy attachment, and OIDC trust policy configuration |
| AWS SSM Parameter Store | Service Discovery | Storage and retrieval of role ARNs for consuming project integration |
| GitHub Actions | CI/CD Automation | Workflow orchestration for deployment, validation, and resource management |
| GitHub OIDC Provider | Authentication | Temporary credential federation eliminating stored secrets |
| AWS S3 | State Backend | Terraform state file storage with versioning and encryption |
| AWS DynamoDB | State Locking | Concurrent deployment protection through distributed locking |
| Bash | Scripting | Automation scripts for bootstrap, deployment, and project creation |
| JSON | Policy Configuration | IAM policy document definition and validation |
| Git | Version Control | Source code management and repository metadata extraction |
| AWS CLI | Command Line Interface | AWS service interaction and credential management |
| jq | JSON Processing | Command-line JSON parsing and manipulation in scripts |

© 2025 Stephen Abbot - MIT License
