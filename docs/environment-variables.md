# Environment Variables Reference

This document serves as the single source of truth for all environment variables used in the eBay Sniper application.

## AWS Configuration

### Core AWS Settings
| Variable | Description | Default Value | Required/Optional | Used By | Environment Scope |
|----------|-------------|---------------|-------------------|---------|-------------------|
| `AWS_REGION` | AWS region for all services | `us-east-1` | Optional | All Lambda functions, Frontend | All |
| `AWS_PROFILE` | AWS CLI profile for authentication | `default` | Optional | Development tools | Development |
| `ENVIRONMENT` | Deployment environment identifier | - | Required | All Lambda functions | All |

### DynamoDB Tables
| Variable | Description | Default Value | Required/Optional | Used By | Environment Scope |
|----------|-------------|---------------|-------------------|---------|-------------------|
| `USERS_TABLE` | DynamoDB Users table name | - | Required | User Management, eBay OAuth, Wishlist Sync, Bid Executor, Notification Handler, Token Refresh, Price Update Lambdas | All |
| `BIDS_TABLE` | DynamoDB Bids table name | - | Required | Bid Management, Bid Executor, Price Update Lambdas | All |
| `BID_HISTORY_TABLE` | DynamoDB Bid History table name | - | Required | Bid Management, Bid History, Bid Executor Lambdas | All |
| `DYNAMODB_ENDPOINT` | DynamoDB endpoint URL for local development | `http://localhost:8000` | Optional | All Lambda functions | Development |

### AWS Secrets Manager
| Variable | Description | Default Value | Required/Optional | Used By | Environment Scope |
|----------|-------------|---------------|-------------------|---------|-------------------|
| `SECRET_ARN` | AWS Secrets Manager ARN for eBay credentials | - | Required | eBay OAuth, Wishlist Sync, Bid Executor, Notification Handler, Token Refresh, Price Update Lambdas | All |

### AWS Cognito
| Variable | Description | Default Value | Required/Optional | Used By | Environment Scope |
|----------|-------------|---------------|-------------------|---------|-------------------|
| `NEXT_PUBLIC_USER_POOL_ID` | AWS Cognito User Pool ID | - | Required | Frontend | All |
| `NEXT_PUBLIC_USER_POOL_CLIENT_ID` | AWS Cognito User Pool Client ID | - | Required | Frontend | All |
| `NEXT_PUBLIC_AWS_REGION` | AWS region for Cognito | `us-east-1` | Required | Frontend | All |

### EventBridge Scheduler
| Variable | Description | Default Value | Required/Optional | Used By | Environment Scope |
|----------|-------------|---------------|-------------------|---------|-------------------|
| `SCHEDULER_GROUP_NAME` | EventBridge scheduler group name | - | Required | Bid Management Lambda | All |
| `BID_EXECUTOR_FUNCTION_ARN` | Target Lambda function ARN for bid execution | - | Required | Bid Management Lambda | All |
| `NOTIFICATION_FUNCTION_NAME` | Notification Handler Lambda function name | - | Required | Bid Executor, Price Update Lambdas | All |

## eBay API Configuration

### eBay Credentials
| Variable | Description | Default Value | Required/Optional | Used By | Environment Scope |
|----------|-------------|---------------|-------------------|---------|-------------------|
| `EBAY_APP_ID` | eBay API Application ID | - | Required | All Lambda functions (via Secrets Manager) | All |
| `EBAY_DEV_ID` | eBay Developer ID | - | Required | Stored in Secrets Manager | All |
| `EBAY_CERT_ID` | eBay Certificate ID | - | Required | Stored in Secrets Manager | All |
| `EBAY_ENVIRONMENT` | eBay API environment (sandbox/production) | `sandbox` | Required | eBay OAuth, Wishlist Sync, Price Update Lambdas | All |
| `EBAY_REDIRECT_URI` | OAuth redirect URI for eBay authentication | - | Required | eBay OAuth Lambda | All |

### eBay Frontend Configuration
| Variable | Description | Default Value | Required/Optional | Used By | Environment Scope |
|----------|-------------|---------------|-------------------|---------|-------------------|
| `NEXT_PUBLIC_EBAY_ENVIRONMENT` | eBay environment for frontend | `sandbox` | Required | Frontend | All |

## Email Service Configuration

### Postmark Integration
| Variable | Description | Default Value | Required/Optional | Used By | Environment Scope |
|----------|-------------|---------------|-------------------|---------|-------------------|
| `POSTMARK_API_KEY` | Postmark API key for email notifications | - | Required | Stored in Secrets Manager, used by Notification Handler Lambda | All |

## Application Configuration

### Development & Debugging
| Variable | Description | Default Value | Required/Optional | Used By | Environment Scope |
|----------|-------------|---------------|-------------------|---------|-------------------|
| `DEBUG` | Enable debug mode | `false` | Optional | All Lambda functions | All |
| `LOG_LEVEL` | Logging level (DEBUG, INFO, WARN, ERROR) | `INFO` | Optional | All Lambda functions | All |

### CORS & Security
| Variable | Description | Default Value | Required/Optional | Used By | Environment Scope |
|----------|-------------|---------------|-------------------|---------|-------------------|
| `CORS_ORIGINS` | Allowed CORS origins | `http://localhost:3000` | Optional | All API Lambda functions | All |

### Token Management
| Variable | Description | Default Value | Required/Optional | Used By | Environment Scope |
|----------|-------------|---------------|-------------------|---------|-------------------|
| `TOKEN_EXPIRY_THRESHOLD` | Days before expiry to refresh tokens | `7` | Optional | Token Refresh Lambda | All |

## Frontend Configuration

### API Integration
| Variable | Description | Default Value | Required/Optional | Used By | Environment Scope |
|----------|-------------|---------------|-------------------|---------|-------------------|
| `NEXT_PUBLIC_API_URL` | Backend API base URL | `http://localhost:8000` | Required | Frontend | All |

### Feature Flags
| Variable | Description | Default Value | Required/Optional | Used By | Environment Scope |
|----------|-------------|---------------|-------------------|---------|-------------------|
| `NEXT_PUBLIC_ENABLE_ANALYTICS` | Enable analytics tracking | `false` | Optional | Frontend | All |
| `NEXT_PUBLIC_ENABLE_DEBUG` | Enable frontend debug mode | `false` | Optional | Frontend | All |

### Amplify Configuration
| Variable | Description | Default Value | Required/Optional | Used By | Environment Scope |
|----------|-------------|---------------|-------------------|---------|-------------------|
| `AMPLIFY_DIFF_DEPLOY` | Enable differential deployment | `false` | Optional | AWS Amplify | All |

## Lambda Function Specific Variables

### Individual Lambda Function Environment Variables
Each Lambda function has access to global environment variables plus specific ones:

#### User Management Lambda
- `USERS_TABLE`
- `CORS_ORIGINS`

#### eBay OAuth Lambda
- `USERS_TABLE`
- `SECRET_ARN`
- `EBAY_APP_ID`
- `EBAY_REDIRECT_URI`
- `EBAY_ENVIRONMENT`

#### Wishlist Sync Lambda
- `USERS_TABLE`
- `SECRET_ARN`
- `EBAY_ENVIRONMENT`

#### Bid Management Lambda
- `BIDS_TABLE`
- `BID_HISTORY_TABLE`
- `SCHEDULER_GROUP_NAME`
- `BID_EXECUTOR_FUNCTION_ARN`

#### Bid History Lambda
- `BID_HISTORY_TABLE`

#### Bid Executor Lambda
- `BIDS_TABLE`
- `BID_HISTORY_TABLE`
- `USERS_TABLE`
- `SECRET_ARN`
- `NOTIFICATION_FUNCTION_NAME`

#### Notification Handler Lambda
- `USERS_TABLE`
- `SECRET_ARN`

#### Token Refresh Lambda
- `USERS_TABLE`
- `SECRET_ARN`
- `TOKEN_EXPIRY_THRESHOLD`

#### Price Update Lambda
- `BIDS_TABLE`
- `USERS_TABLE`
- `SECRET_ARN`
- `NOTIFICATION_FUNCTION_NAME`
- `EBAY_ENVIRONMENT`

## Environment-Specific Values

### Development Environment (.env.dev)
```bash
ENVIRONMENT=dev
DEBUG=true
LOG_LEVEL=DEBUG
EBAY_ENVIRONMENT=sandbox
AWS_REGION=us-east-1
AWS_PROFILE=default
```

### Staging Environment (.env.staging)
```bash
ENVIRONMENT=staging
DEBUG=false
LOG_LEVEL=INFO
EBAY_ENVIRONMENT=sandbox
AWS_REGION=us-east-1
```

### Production Environment (.env.prod)
```bash
ENVIRONMENT=prod
DEBUG=false
LOG_LEVEL=INFO
EBAY_ENVIRONMENT=production
AWS_REGION=us-east-1
AWS_PROFILE=production
```

## Security Notes

### Secrets Management
The following sensitive variables are stored in AWS Secrets Manager:
- `EBAY_APP_ID`
- `EBAY_DEV_ID`
- `EBAY_CERT_ID`
- `POSTMARK_API_KEY`

### DynamoDB Security
- All table names are generated dynamically using CloudFormation stack names
- Encryption at rest is enabled automatically
- IAM policies enforce user-level data isolation

### Frontend Security
- All public environment variables use the `NEXT_PUBLIC_` prefix
- Sensitive backend variables are never exposed to the frontend
- API authentication is handled via AWS Cognito JWT tokens

## Usage Notes

### For Developers
- Reference this document when adding new environment variables
- Always use consistent naming conventions
- Update this document when adding or modifying variables

### For DevOps
- Use this as the authoritative source for environment configuration
- Ensure all environments have required variables configured
- Validate variable values during deployment

### For Documentation
- Other documentation files should reference this document instead of duplicating variable definitions
- Keep individual service documentation focused on functionality, not configuration details