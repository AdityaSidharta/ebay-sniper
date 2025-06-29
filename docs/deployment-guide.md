# Deployment Guide

This guide provides step-by-step instructions for deploying the eBay Sniper application to AWS environments.

## Prerequisites

### Required Tools
- **AWS CLI v2**: Configure with appropriate credentials
- **AWS SAM CLI**: Version 1.100.0 or later
- **Node.js**: Version 18.x or later (for frontend)
- **Python**: Version 3.11 or later (for backend)
- **Git**: For source code management
- **pnpm**: Package manager for frontend dependencies

### AWS Account Setup
- AWS account with appropriate permissions
- AWS CLI configured with credentials
- S3 bucket for deployment artifacts
- Domain name registered (optional, for custom domains)

### Required Permissions
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "InfrastructureManagement",
      "Effect": "Allow",
      "Action": [
        "cloudformation:*",
        "lambda:*",
        "apigateway:*",
        "dynamodb:*",
        "logs:*"
      ],
      "Resource": [
        "arn:aws:cloudformation:*:*:stack/ebay-sniper-*/*",
        "arn:aws:cloudformation:*:aws:transform/Serverless-*",
        "arn:aws:lambda:*:*:function:ebay-sniper-*",
        "arn:aws:apigateway:*::/restapis*",
        "arn:aws:dynamodb:*:*:table/ebay-sniper-*",
        "arn:aws:logs:*:*:log-group:/aws/lambda/ebay-sniper-*"
      ]
    },
    {
      "Sid": "AuthenticationAndSecrets",
      "Effect": "Allow",
      "Action": [
        "cognito-idp:*",
        "secretsmanager:*",
        "iam:CreateRole",
        "iam:DeleteRole",
        "iam:GetRole",
        "iam:PassRole",
        "iam:AttachRolePolicy",
        "iam:DetachRolePolicy",
        "iam:PutRolePolicy",
        "iam:DeleteRolePolicy",
        "iam:TagRole",
        "iam:UntagRole"
      ],
      "Resource": [
        "arn:aws:cognito-idp:*:*:userpool/*",
        "arn:aws:secretsmanager:*:*:secret:ebay-sniper-*",
        "arn:aws:iam::*:role/ebay-sniper-*"
      ]
    },
    {
      "Sid": "SchedulingAndHosting",
      "Effect": "Allow",
      "Action": [
        "events:*",
        "scheduler:*",
        "amplify:*",
        "s3:*"
      ],
      "Resource": [
        "arn:aws:events:*:*:rule/ebay-sniper-*",
        "arn:aws:scheduler:*:*:schedule-group/ebay-sniper-*",
        "arn:aws:amplify:*:*:apps/*/branches/*",
        "arn:aws:s3:::ebay-sniper-*",
        "arn:aws:s3:::ebay-sniper-*/*"
      ]
    }
  ]
}
```

## Environment Setup

### 1. Clone Repository
```bash
git clone https://github.com/your-org/ebay-sniper.git
cd ebay-sniper
```

### 2. Backend Dependencies
```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
pip install -r requirements-dev.txt
```

### 3. Frontend Dependencies
```bash
cd frontend
pnpm install
cd ..
```

### 4. Environment Variables
Create environment-specific configuration files:

#### .env.dev
```bash
# Development Environment
ENVIRONMENT=dev
DEBUG=true
LOG_LEVEL=DEBUG

# eBay API (Sandbox)
EBAY_ENVIRONMENT=sandbox
EBAY_APP_ID=your_dev_app_id
EBAY_DEV_ID=your_dev_id
EBAY_CERT_ID=your_dev_cert_id

# Postmark
POSTMARK_API_KEY=your_dev_postmark_key

# AWS
AWS_REGION=us-east-1
AWS_PROFILE=default
```

#### .env.prod
```bash
# Production Environment
ENVIRONMENT=prod
DEBUG=false
LOG_LEVEL=INFO

# eBay API (Production)
EBAY_ENVIRONMENT=production
EBAY_APP_ID=your_prod_app_id
EBAY_DEV_ID=your_prod_dev_id
EBAY_CERT_ID=your_prod_cert_id

# Postmark
POSTMARK_API_KEY=your_prod_postmark_key

# AWS
AWS_REGION=us-east-1
AWS_PROFILE=production
```

## Pre-Deployment Validation

### 1. Code Quality Checks
```bash
# Backend linting and testing
cd backend
# Code formatting and linting
ruff format src/ tests/
ruff check src/ tests/

# Type checking
ty src/
pytest tests/ -v

# Frontend linting and testing
cd ../frontend
pnpm lint
pnpm type-check
pnpm test
cd ..
```

### 2. Security Scanning
```bash
# Python security scan
pip install bandit
bandit -r src/

# Node.js security audit
cd frontend
pnpm audit
cd ..
```

### 3. Configuration Validation
```bash
# Validate SAM template
sam validate --template template.yaml

# Check environment variables
python scripts/validate-config.py --env dev
```

## Deployment Process

### Development Environment

#### 1. Build Application
```bash
# Build SAM application
sam build --use-container

# Build frontend
cd frontend
pnpm build
cd ..
```

#### 2. Deploy Infrastructure
```bash
# Deploy backend infrastructure
sam deploy \
  --config-env dev \
  --parameter-overrides Environment=dev \
  --capabilities CAPABILITY_IAM \
  --confirm-changeset

# Get stack outputs
aws cloudformation describe-stacks \
  --stack-name ebay-sniper-dev \
  --query 'Stacks[0].Outputs'
```

#### 3. Configure Secrets
```bash
# Update eBay credentials in Secrets Manager
aws secretsmanager update-secret \
  --secret-id ebay-sniper-dev/ebay-credentials \
  --secret-string '{
    "app_id": "your_dev_app_id",
    "dev_id": "your_dev_id",
    "cert_id": "your_dev_cert_id",
    "environment": "sandbox"
  }'
```

#### 4. Deploy Frontend
```bash
# Configure Amplify app
cd frontend

# Set environment variables
export NEXT_PUBLIC_API_URL=$(aws cloudformation describe-stacks \
  --stack-name ebay-sniper-dev \
  --query 'Stacks[0].Outputs[?OutputKey==`ApiUrl`].OutputValue' \
  --output text)

export NEXT_PUBLIC_USER_POOL_ID=$(aws cloudformation describe-stacks \
  --stack-name ebay-sniper-dev \
  --query 'Stacks[0].Outputs[?OutputKey==`UserPoolId`].OutputValue' \
  --output text)

export NEXT_PUBLIC_USER_POOL_CLIENT_ID=$(aws cloudformation describe-stacks \
  --stack-name ebay-sniper-dev \
  --query 'Stacks[0].Outputs[?OutputKey==`UserPoolClientId`].OutputValue' \
  --output text)

# Deploy to Amplify
git add .
git commit -m "Deploy to dev environment"
git push origin develop
```

### Staging Environment

#### 1. Merge and Tag
```bash
# Merge develop to staging branch
git checkout staging
git merge develop
git tag -a v1.0.0-staging -m "Staging release v1.0.0"
git push origin staging --tags
```

#### 2. Deploy to Staging
```bash
# Deploy infrastructure
sam deploy \
  --config-env staging \
  --parameter-overrides Environment=staging \
  --capabilities CAPABILITY_IAM \
  --no-confirm-changeset

# Update secrets
aws secretsmanager update-secret \
  --secret-id ebay-sniper-staging/ebay-credentials \
  --secret-string '{
    "app_id": "your_staging_app_id",
    "dev_id": "your_staging_dev_id",
    "cert_id": "your_staging_cert_id",
    "environment": "sandbox"
  }'
```

#### 3. Staging Tests
```bash
# Run integration tests against staging
python scripts/integration-tests.py --env staging

# Load testing
python scripts/load-test.py --env staging --duration 300
```

### Production Environment

#### 1. Create Release
```bash
# Merge staging to main
git checkout main
git merge staging
git tag -a v1.0.0 -m "Production release v1.0.0"
git push origin main --tags
```

#### 2. Production Deployment
```bash
# Deploy infrastructure with extra safety
sam deploy \
  --config-env prod \
  --parameter-overrides Environment=prod \
  --capabilities CAPABILITY_IAM \
  --no-fail-on-empty-changeset \
  --confirm-changeset

# Update production secrets
aws secretsmanager update-secret \
  --secret-id ebay-sniper-prod/ebay-credentials \
  --secret-string '{
    "app_id": "your_prod_app_id",
    "dev_id": "your_prod_dev_id",
    "cert_id": "your_prod_cert_id",
    "environment": "production"
  }'
```

#### 3. Post-Deployment Verification
```bash
# Health check
curl -X GET https://api.ebay-sniper.com/health

# Smoke tests
python scripts/smoke-tests.py --env prod

# Monitor deployment
aws logs tail /aws/lambda/ebay-sniper-prod-api --follow
```

## Database Migration

### Initial Setup
```bash
# Create tables and indexes
python scripts/db-setup.py --env dev

# Seed test data (dev only)
python scripts/seed-data.py --env dev
```

### Schema Updates
```bash
# Run migration script
python scripts/migrate-schema.py --env prod --dry-run
python scripts/migrate-schema.py --env prod --confirm
```

## Configuration Management

### Parameter Store
```bash
# Set application parameters
aws ssm put-parameter \
  --name "/ebay-sniper/prod/max-bid-amount" \
  --value "10000000" \
  --type "String"

aws ssm put-parameter \
  --name "/ebay-sniper/prod/rate-limit-per-minute" \
  --value "100" \
  --type "String"
```

### Feature Flags
```bash
# Enable/disable features
aws ssm put-parameter \
  --name "/ebay-sniper/prod/feature/email-notifications" \
  --value "true" \
  --type "String" \
  --overwrite
```

## Monitoring Setup

### CloudWatch Dashboards
```bash
# Deploy monitoring stack
aws cloudformation deploy \
  --template-file monitoring/dashboard.yaml \
  --stack-name ebay-sniper-monitoring-prod \
  --parameter-overrides Environment=prod
```

### Alerts Configuration
```bash
# Create SNS topic for alerts
aws sns create-topic --name ebay-sniper-alerts-prod

# Subscribe to alerts
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:ebay-sniper-alerts-prod \
  --protocol email \
  --notification-endpoint admin@example.com
```

## SSL Certificate Setup

### ACM Certificate
```bash
# Request certificate
aws acm request-certificate \
  --domain-name api.ebay-sniper.com \
  --domain-name ebay-sniper.com \
  --validation-method DNS \
  --region us-east-1

# Get certificate ARN
CERT_ARN=$(aws acm list-certificates \
  --query 'CertificateSummaryList[?DomainName==`api.ebay-sniper.com`].CertificateArn' \
  --output text)
```

### Custom Domain Setup
```bash
# Create custom domain
aws apigatewayv2 create-domain-name \
  --domain-name api.ebay-sniper.com \
  --domain-name-configurations CertificateArn=$CERT_ARN

# Create API mapping
aws apigatewayv2 create-api-mapping \
  --domain-name api.ebay-sniper.com \
  --api-id $API_ID \
  --stage $STAGE
```

## Rollback Procedures

### Infrastructure Rollback
```bash
# Rollback to previous stack version
aws cloudformation cancel-update-stack \
  --stack-name ebay-sniper-prod

# Or rollback to specific changeset
aws cloudformation execute-change-set \
  --change-set-name previous-changeset-arn
```

### Application Rollback
```bash
# Rollback Lambda function code
aws lambda update-function-code \
  --function-name ebay-sniper-prod-api \
  --s3-bucket deployment-bucket \
  --s3-key previous-deployment.zip

# Rollback frontend
git checkout previous-release-tag
cd frontend
pnpm build
# Deploy to Amplify
```

### Database Rollback
```bash
# Point-in-time recovery
aws dynamodb restore-table-to-point-in-time \
  --source-table-name ebay-sniper-prod-Users \
  --target-table-name ebay-sniper-prod-Users-restored \
  --restore-date-time 2024-01-01T12:00:00Z
```

## Troubleshooting

### Common Deployment Issues

#### 1. Permission Errors
```bash
# Check IAM permissions
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/deployer \
  --action-names cloudformation:CreateStack \
  --resource-arns "*"
```

#### 2. Stack Update Failures
```bash
# Check stack events
aws cloudformation describe-stack-events \
  --stack-name ebay-sniper-prod \
  --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`]'

# View detailed error
aws logs filter-log-events \
  --log-group-name /aws/cloudformation/ebay-sniper-prod \
  --start-time $(date -d '1 hour ago' +%s)000
```

#### 3. Lambda Function Issues
```bash
# Check function configuration
aws lambda get-function-configuration \
  --function-name ebay-sniper-prod-api

# View recent logs
aws logs tail /aws/lambda/ebay-sniper-prod-api \
  --since 1h \
  --follow
```

#### 4. DynamoDB Access Issues
```bash
# Test table access
aws dynamodb describe-table \
  --table-name ebay-sniper-prod-Users

# Check table status
aws dynamodb wait table-exists \
  --table-name ebay-sniper-prod-Users
```

### Performance Issues

#### 1. Cold Start Optimization
```bash
# Enable provisioned concurrency
aws lambda put-provisioned-concurrency-config \
  --function-name ebay-sniper-prod-api \
  --qualifier $LATEST \
  --provisioned-concurrency-configs 10
```

#### 2. DynamoDB Throttling
```bash
# Check metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name UserErrors \
  --dimensions Name=TableName,Value=ebay-sniper-prod-Users \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --period 300 \
  --statistics Sum
```

## CI/CD Pipeline

### GitHub Actions Workflow
```yaml
name: Deploy eBay Sniper
on:
  push:
    branches: [main, staging, develop]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: aws-actions/setup-sam@v2
      - uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Determine environment
        id: env
        run: |
          if [[ $GITHUB_REF == 'refs/heads/main' ]]; then
            echo "environment=prod" >> $GITHUB_OUTPUT
          elif [[ $GITHUB_REF == 'refs/heads/staging' ]]; then
            echo "environment=staging" >> $GITHUB_OUTPUT
          else
            echo "environment=dev" >> $GITHUB_OUTPUT
          fi
      
      - name: SAM build and deploy
        run: |
          sam build
          sam deploy --config-env ${{ steps.env.outputs.environment }}
```

## Security Checklist

### Pre-Deployment Security Review
- [ ] Secrets stored in AWS Secrets Manager
- [ ] IAM roles follow least privilege principle
- [ ] KMS encryption enabled for sensitive data
- [ ] API Gateway throttling configured
- [ ] WAF rules implemented
- [ ] Security groups restrict access
- [ ] CloudTrail logging enabled
- [ ] VPC endpoints configured for AWS services

### Post-Deployment Security Validation
- [ ] Run security scanner (AWS Inspector)
- [ ] Test authentication and authorization
- [ ] Verify encryption at rest and in transit
- [ ] Check access logs for anomalies
- [ ] Validate backup and recovery procedures
- [ ] Review IAM access patterns

## Disaster Recovery

### Backup Verification
```bash
# Verify DynamoDB backups
aws dynamodb list-backups \
  --table-name ebay-sniper-prod-Users \
  --backup-type USER

# Test restore procedure
aws dynamodb restore-table-from-backup \
  --target-table-name ebay-sniper-test-restore \
  --backup-arn arn:aws:dynamodb:us-east-1:123456789012:table/Users/backup/01234567890123-abcdefgh
```

### Cross-Region Setup
```bash
# Deploy to secondary region
sam deploy \
  --config-env prod-dr \
  --region us-west-2 \
  --parameter-overrides Environment=prod-dr
```

## Maintenance

### Regular Maintenance Tasks
- Weekly: Review CloudWatch logs and metrics
- Monthly: Update dependencies and security patches
- Quarterly: Review and rotate secrets
- Annually: Disaster recovery testing

### Health Monitoring
```bash
# Create health check script
curl -f https://api.ebay-sniper.com/health || exit 1

# Schedule in cron
# */5 * * * * /path/to/health-check.sh
```