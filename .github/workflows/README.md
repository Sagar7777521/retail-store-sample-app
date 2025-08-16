# GitHub Workflows for Retail Store Application

This directory contains GitHub Actions workflows for automated CI/CD pipeline that detects changes, builds Docker images, pushes them to ECR, and updates Helm chart values.

## Workflows

### 1. CI/CD Pipeline (`ci-cd.yml`)
**Purpose**: Main workflow for building and deploying microservices

**Triggers**:
- Push to `main` or `develop` branches with changes in `src/` directory
- Pull requests to `main` branch with changes in `src/` directory

**Features**:
- **Change Detection**: Only builds services that have been modified
- **Multi-service Build**: Supports all 5 microservices (ui, cart, catalog, checkout, orders)
- **ECR Integration**: Automatically creates ECR repositories if they don't exist
- **Helm Updates**: Updates Helm chart values with new image tags
- **GitOps Ready**: Commits updated values back to repository for ArgoCD sync

### 2. Infrastructure Management (`infrastructure.yml`)
**Purpose**: Manages Terraform infrastructure including ECR repositories

**Triggers**:
- Push to `main` branch with changes in `terraform/` directory
- Manual workflow dispatch with plan/apply/destroy options

## Setup Instructions

### 1. Required GitHub Secrets

Go to your GitHub repository → Settings → Secrets and variables → Actions → New repository secret.

| Secret Name | Description | Example Value |
|-------------|-------------|---------------|
| `AWS_ACCESS_KEY_ID` | AWS Access Key ID | `AKIAIOSFODNN7EXAMPLE` |
| `AWS_SECRET_ACCESS_KEY` | AWS Secret Access Key | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY` |
| `AWS_REGION` | AWS Region | `us-west-2` |
| `AWS_ACCOUNT_ID` | AWS Account ID | `123456789012` |

### 2. IAM Permissions

The AWS user/role needs the following permissions:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ecr:GetAuthorizationToken",
                "ecr:BatchCheckLayerAvailability",
                "ecr:GetDownloadUrlForLayer",
                "ecr:BatchGetImage",
                "ecr:BatchImportLayerPart",
                "ecr:CompleteLayerUpload",
                "ecr:DescribeRepositories",
                "ecr:CreateRepository",
                "ecr:InitiateLayerUpload",
                "ecr:PutImage",
                "ecr:UploadLayerPart"
            ],
            "Resource": "*"
        }
    ]
}
```

### 3. Repository Structure

Ensure your repository has the following structure:
```
├── src/
│   ├── ui/
│   │   ├── Dockerfile
│   │   └── chart/values.yaml
│   ├── cart/
│   │   ├── Dockerfile
│   │   └── chart/values.yaml
│   ├── catalog/
│   │   ├── Dockerfile
│   │   └── chart/values.yaml
│   ├── checkout/
│   │   ├── Dockerfile
│   │   └── chart/values.yaml
│   └── orders/
│       ├── Dockerfile
│       └── chart/values.yaml
├── argocd/
│   └── applications/
│       ├── retail-store-ui.yaml
│       ├── retail-store-cart.yaml
│       ├── retail-store-catalog.yaml
│       ├── retail-store-checkout.yaml
│       └── retail-store-orders.yaml
└── terraform/
    ├── main.tf
    ├── ecr.tf
    └── ...
```

## How It Works

### 1. Change Detection
The workflow uses `dorny/paths-filter` to detect which services have changed:
- Only modified services are built and deployed
- Reduces build time and resource usage
- Prevents unnecessary deployments

### 2. Docker Build & Push
For each changed service:
- Builds Docker image using the service's Dockerfile
- Tags with both commit SHA and 'latest'
- Creates ECR repository if it doesn't exist
- Pushes images to ECR

### 3. Helm Values Update
- Updates `values.yaml` files with new image tags
- Updates repository URLs to point to private ECR
- Commits changes back to repository with `[skip ci]` to prevent loops

### 4. ArgoCD Integration
- ArgoCD monitors the repository for changes
- Automatically syncs applications when Helm values are updated
- Provides GitOps workflow with full audit trail

## Workflow Outputs

### Successful Build
```
✅ Deployment successful! Images built and Helm values updated.
🚀 ArgoCD will automatically sync the changes.
```

### ECR Repositories Created
- `retail-store-ui`
- `retail-store-cart`
- `retail-store-catalog`
- `retail-store-checkout`
- `retail-store-orders`

### Updated Files
- `src/*/chart/values.yaml` - Updated with new image tags
- `argocd/applications/*.yaml` - Updated with new target revisions

## Troubleshooting

### Common Issues

1. **ECR Permission Denied**
   - Verify AWS credentials are correct
   - Check IAM permissions include ECR actions
   - Ensure AWS_ACCOUNT_ID secret is set correctly

2. **Docker Build Fails**
   - Check Dockerfile syntax in the service directory
   - Verify all required files are present
   - Review build logs for specific errors

3. **Helm Values Not Updated**
   - Ensure `values.yaml` files exist in `src/*/chart/` directories
   - Check file permissions and Git configuration
   - Verify the sed commands match your values.yaml structure

4. **ArgoCD Not Syncing**
   - Check ArgoCD application configurations
   - Verify repository URL and credentials
   - Ensure target revision is set correctly

### Manual Triggers

You can manually trigger workflows:

1. **CI/CD Pipeline**: Push changes to `src/` directory
2. **Infrastructure**: Use workflow dispatch in GitHub Actions tab

### Monitoring

Monitor workflow execution in:
- GitHub Actions tab in your repository
- ECR console for pushed images
- ArgoCD UI for application sync status

## Customization

### Adding New Services
1. Create service directory under `src/`
2. Add Dockerfile and Helm chart
3. Update the `services` list in `terraform/ecr.tf`
4. Add change detection filter in workflow

### Changing Regions
1. Update `AWS_REGION` secret
2. Update `ECR_REGISTRY` environment variable
3. Update Terraform region configuration

### Custom Image Tags
Modify the `IMAGE_TAG` environment variable in the workflow to use different tagging strategies (e.g., semantic versioning, branch names).