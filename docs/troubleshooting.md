# Troubleshooting

## GitHub Actions Authentication Issues

### AssumeRoleWithWebIdentity Failures

**Problem**: Workflow fails with "AssumeRoleWithWebIdentity" error

**Root Causes and Solutions**:

1. **OIDC Provider Missing**
   ```bash
   # Verify foundation infrastructure deployed
   aws ssm get-parameter --name "/terraform/foundation/oidc-provider"
   ```
   - Deploy [foundation-terraform-bootstrap](https://github.com/stephenabbot/foundation-terraform-bootstrap) if parameter missing
   - Verify OIDC provider exists in IAM console

2. **AWS_ACCOUNT_ID Variable Missing**
   ```bash
   # Check repository variable exists
   gh variable list | grep AWS_ACCOUNT_ID
   
   # Set if missing
   ./scripts/bootstrap.sh
   ```

3. **Repository Name Mismatch**
   - Verify role trust policy includes correct repository name
   - Repository names are case-sensitive in trust policies
   - Check role trust policy in IAM console matches GitHub repository

### Workflow Permission Issues

**Problem**: "Token request failed" or permission denied errors

**Solutions**:
```yaml
# Ensure workflow has correct permissions
permissions:
  id-token: write  # Required for OIDC token
  contents: read   # Required for checkout
```

**Problem**: "Parameter not found" errors in workflows

**Root Causes**:
1. Foundation project SSM parameters missing
2. Foundation deployment role lacks SSM permissions
3. Incorrect parameter paths

**Solutions**:
```bash
# Verify foundation parameters exist
aws ssm get-parameters-by-path --path "/terraform/foundation" --recursive

# Check specific parameters
aws ssm get-parameter --name "/terraform/foundation/s3-state-bucket"
aws ssm get-parameter --name "/terraform/foundation/dynamodb-lock-table"
aws ssm get-parameter --name "/terraform/foundation/oidc-provider"
```

## Workflow Visibility Issues

### Destroy Workflow Not Visible

**Problem**: Destroy workflow not visible in Actions tab

**Solution**: This is normal behavior for `workflow_dispatch` workflows
- Go to Actions → "All workflows" sidebar → Find "Destroy"
- Workflows with manual triggers are hidden until first execution
- This is a safety feature to prevent accidental resource destruction

### Workflow Status Confusion

**Problem**: Workflow appears successful but resources not created

**Diagnostic Steps**:
```bash
# Check OpenTofu state
./scripts/list-deployed-resources.sh

# Verify role creation
aws iam list-roles --query 'Roles[?starts_with(RoleName, `gharole-`)]'

# Check SSM parameters
aws ssm get-parameters-by-path --path "/deployment-roles" --recursive
```

## Backend Configuration Issues

### Backend Initialization Failures

**Problem**: "Backend initialization failed" during workflow

**Root Causes and Solutions**:

1. **S3 Bucket Missing**
   ```bash
   # Verify bucket exists
   BUCKET=$(aws ssm get-parameter --name "/terraform/foundation/s3-state-bucket" --query Parameter.Value --output text)
   aws s3 ls "s3://$BUCKET"
   ```

2. **DynamoDB Table Missing**
   ```bash
   # Verify table exists
   TABLE=$(aws ssm get-parameter --name "/terraform/foundation/dynamodb-lock-table" --query Parameter.Value --output text)
   aws dynamodb describe-table --table-name "$TABLE"
   ```

3. **Permission Issues**
   - Verify foundation role has S3 and DynamoDB permissions
   - Check bucket policy allows foundation role access
   - Ensure DynamoDB table allows read/write operations

### State Lock Issues

**Problem**: "Error acquiring the state lock" or "Lock Info" errors

**Automatic Resolution**:
- Deploy script automatically clears stale locks older than 1 hour
- Manual intervention rarely required

**Manual Resolution**:
```bash
# Check for locks in DynamoDB
TABLE=$(aws ssm get-parameter --name "/terraform/foundation/dynamodb-lock-table" --query Parameter.Value --output text)
aws dynamodb scan --table-name "$TABLE"

# Force unlock if necessary (use with caution)
tofu force-unlock <LOCK_ID>
```

## Role Assumption Failures

### Trust Policy Issues

**Problem**: GitHub Actions cannot assume deployment role

**Diagnostic Steps**:
```bash
# Get role trust policy
aws iam get-role --role-name gharole-my-project-prd --query Role.AssumeRolePolicyDocument

# Verify OIDC provider URL matches
aws iam get-open-id-connect-provider --open-id-connect-provider-arn <OIDC_ARN>
```

**Common Issues**:
1. **Repository Name Mismatch** - Trust policy must match exact repository name
2. **OIDC Provider URL** - Must match between trust policy and actual provider
3. **Audience Mismatch** - Must be "sts.amazonaws.com"

### Role Not Found Errors

**Problem**: Role ARN not found during assumption

**Solutions**:
```bash
# Verify role exists
aws iam get-role --role-name gharole-my-project-prd

# Check role was created by deployment
./scripts/list-deployed-resources.sh

# Verify SSM parameter contains correct ARN
aws ssm get-parameter --name "/deployment-roles/my-project/role-arn"
```

## Policy Issues

### Invalid Policy JSON

**Problem**: Deployment fails with policy validation errors

**Solutions**:
```bash
# Validate JSON syntax
jq . projects/my-project/prd/policies/deployment-policy.json

# Check policy document size (max 6,144 characters for inline policies)
wc -c projects/my-project/prd/policies/deployment-policy.json

# Test policy with AWS Policy Simulator
aws iam simulate-principal-policy --policy-source-arn <ROLE_ARN> --action-names <ACTION> --resource-arns <RESOURCE>
```

### Policy Not Attached

**Problem**: Role exists but lacks permissions

**Diagnostic Steps**:
```bash
# List attached policies
aws iam list-attached-role-policies --role-name gharole-my-project-prd

# Verify policy exists
aws iam get-policy --policy-arn <POLICY_ARN>

# Check OpenTofu state shows attachment
tofu state list | grep policy_attachment
```

### Insufficient Permissions

**Problem**: Role has policy but operations still fail

**Solutions**:
1. **Start Broad, Refine Later**
   - Begin with broad permissions for testing
   - Use CloudTrail to identify required permissions
   - Gradually restrict to least privilege

2. **CloudTrail Analysis**
   ```bash
   # Find denied actions in CloudTrail
   aws logs filter-log-events --log-group-name CloudTrail/IAMDeniedActions --filter-pattern "{ $.errorCode = \"AccessDenied\" }"
   ```

## SSM Parameter Access Issues

### Parameter Not Found

**Problem**: Consuming projects cannot find role ARN

**Diagnostic Steps**:
```bash
# Verify parameter exists
aws ssm get-parameter --name "/deployment-roles/my-project/role-arn"

# Check parameter region (must be us-east-1)
aws ssm get-parameter --region us-east-1 --name "/deployment-roles/my-project/role-arn"

# List all deployment role parameters
aws ssm get-parameters-by-path --path "/deployment-roles" --recursive
```

### Parameter Permissions

**Problem**: Access denied when reading parameters

**Solutions**:
```bash
# Verify consuming project has SSM permissions
# Policy should include:
{
  "Effect": "Allow",
  "Action": ["ssm:GetParameter"],
  "Resource": "arn:aws:ssm:us-east-1:*:parameter/deployment-roles/*"
}

# Test parameter access
aws ssm get-parameter --name "/deployment-roles/my-project/role-arn"
```

## Deployment Script Failures

### Prerequisites Not Met

**Problem**: Deploy script fails with prerequisite errors

**Solutions**:
```bash
# Run prerequisite validation
./scripts/verify-prerequisites.sh

# Common fixes
git status --porcelain  # Ensure clean working directory
git push origin main    # Ensure repository is pushed
aws sts get-caller-identity  # Verify AWS credentials
```

### Git Repository Issues

**Problem**: Script cannot detect repository information

**Solutions**:
```bash
# Verify git remote
git remote get-url origin

# Should return GitHub URL like:
# https://github.com/user/repo.git
# git@github.com:user/repo.git

# Fix if needed
git remote set-url origin https://github.com/user/repo.git
```

## Project Creation Issues

### Self-Deployment Prevention

**Problem**: Cannot create deployment role for current project

**Explanation**: This is intentional to prevent circular dependencies

**Solution**: Use different project name or deploy from different repository

**Example**:
```bash
# This will fail if run from foundation-iam-deploy-roles repository
./scripts/create-project.sh foundation-iam-deploy-roles

# Use different name instead
./scripts/create-project.sh my-application-infrastructure
```

### Role Name Length Limits

**Problem**: Role name exceeds 64 character limit

**Error**: `Role name 'gharole-very-long-project-name-prd' exceeds 64 characters`

**Solution**: Use shorter project names
```bash
# Too long (68 characters)
./scripts/create-project.sh very-long-application-infrastructure-project

# Better (47 characters)
./scripts/create-project.sh app-infrastructure
```

## Diagnostic Commands

### Check Role Status

```bash
# List all deployment roles
aws iam list-roles --query 'Roles[?starts_with(RoleName, `gharole-`)].[RoleName,CreateDate]' --output table

# Get specific role details
aws iam get-role --role-name gharole-my-project-prd

# Check role tags
aws iam list-role-tags --role-name gharole-my-project-prd
```

### Check SSM Parameters

```bash
# List all deployment role parameters
aws ssm get-parameters-by-path --path "/deployment-roles" --recursive --output table

# Get specific parameter with metadata
aws ssm get-parameter --name "/deployment-roles/my-project/role-arn" --with-decryption

# Check parameter tags
aws ssm list-tags-for-resource --resource-type Parameter --resource-id "/deployment-roles/my-project/role-arn"
```

### Test Role Assumption

```bash
# Assume role manually (requires appropriate permissions)
aws sts assume-role \
  --role-arn "arn:aws:iam::123456789012:role/gharole-my-project-prd" \
  --role-session-name "test-session"

# Test with temporary credentials
export AWS_ACCESS_KEY_ID=<temp-key>
export AWS_SECRET_ACCESS_KEY=<temp-secret>
export AWS_SESSION_TOKEN=<temp-token>
aws sts get-caller-identity
```

### Verify Backend Access

```bash
# Test S3 state bucket access
BUCKET=$(aws ssm get-parameter --name "/terraform/foundation/s3-state-bucket" --query Parameter.Value --output text)
aws s3 ls "s3://$BUCKET/deployment-roles/"

# Test DynamoDB lock table access
TABLE=$(aws ssm get-parameter --name "/terraform/foundation/dynamodb-lock-table" --query Parameter.Value --output text)
aws dynamodb describe-table --table-name "$TABLE"
```

## Resource Cleanup

### Complete Cleanup

```bash
# Destroy all resources
./scripts/destroy.sh

# Verify cleanup
./scripts/list-deployed-resources.sh

# Manual cleanup if needed
aws iam list-roles --query 'Roles[?starts_with(RoleName, `gharole-`)].[RoleName]' --output text | \
  xargs -I {} aws iam delete-role --role-name {}
```

### Partial Cleanup

```bash
# Remove specific project
aws iam detach-role-policy --role-name gharole-my-project-prd --policy-arn <POLICY_ARN>
aws iam delete-role --role-name gharole-my-project-prd
aws iam delete-policy --policy-arn <POLICY_ARN>
aws ssm delete-parameter --name "/deployment-roles/my-project/role-arn"
```

### Tag-Based Cleanup

```bash
# Find resources by project tag
aws iam list-roles --query 'Roles[?Tags[?Key==`Project` && Value==`foundation-iam-deploy-roles`]].[RoleName]' --output text

# Use tags for automated cleanup scripts
aws resourcegroupstaggingapi get-resources --tag-filters Key=Project,Values=foundation-iam-deploy-roles
```
