# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an eBay sniper application repository - a tool for automated bidding on eBay auctions. The project is licensed under GNU Affero General Public License v3.

## Key Functions

Using the applications, a user should be able to:

- Create an account
- Sync / Link their Ebay Account
- Select items from their eBay wishlist that are currently active for bidding
- Set a price that they are willing to bid, which we will submit to Ebay 5 seconds before the bidding closes. We will only submit the bidding if the price that the user is willing to bid is lower than the current price 
- Send an email indicating whether the user has successfully won the bid or not
- View the list of proposed bids that they have submitted
- Cancel their proposed bids
- Change the proposed bids price that they are willing to pay
- View their bid history
- Manage Notification preferences (Allowing them to mute their email notifications)

## Technology Stack

This is a full-stack application, where it uses AWS Services in order to host and deploy the application

- AWS Cognito will be used to handle user authentication. See @docs/claude-md-nextjs.md for more details
- AWS Amplify will be used to handle the front-end. See @docs/claude-md-python-lambda.md for more details
- AWS Lambda will be used to handle the backend
- AWS API Gateway will be used as the interface layer
- Amazon DynamoDB will be used as the Database
- Postmark will be used to send email notification to the User
- FastAPI will be used for the API Interface

The Language that will be used for the application is as follows:

- Next.JS, Tailwind, and TypeScript will be used for Front-End Framework
- Python will be used for the backend, handling authentication, connecting to databse, as well as connecting with the External Ebay API

## Legal Considerations

**IMPORTANT**: This project involves automated bidding on eBay auctions. Any implementation must:
- Comply with eBay's Terms of Service and API usage policies
- Respect rate limiting and API guidelines
- Consider the ethical implications of automated bidding systems
- Ensure compliance with relevant laws and regulations

## eBay API Integration

- eBay Developer Account credentials (App ID, Cert ID, Dev ID)
- eBay API environment (Sandbox vs Production)
- OAuth scopes required: `https://api.ebay.com/oauth/api_scope/sell.marketing.readonly https://api.ebay.com/oauth/api_scope/sell.inventory.readonly`
- Rate limiting: 5000 calls per day for production
- Required eBay API endpoints:
  - Trading API for bidding operations
  - Browse API for wishlist retrieval
  - OAuth API for account linking

## Required Environment Variables

### AWS Configuration
- AWS_REGION
- DYNAMODB_TABLE_PREFIX
- COGNITO_USER_POOL_ID
- COGNITO_APP_CLIENT_ID

### eBay API
- EBAY_APP_ID
- EBAY_CERT_ID (store in AWS Secrets Manager)
- EBAY_DEV_ID (store in AWS Secrets Manager)
- EBAY_ENVIRONMENT (sandbox/production)

### Email Service
- POSTMARK_API_KEY (store in AWS Secrets Manager)
- POSTMARK_FROM_EMAIL

### Encryption
- EBAY_TOKENS_KMS_KEY_ID

## Development Environment Setup

### Prerequisites
- Node.js 18.x or later
- Python 3.11+
- AWS CLI configured
- eBay Developer Account

### Local Development
- Use AWS SAM CLI for Lambda development
- Use Next.js dev server for frontend
- DynamoDB Local for database testing
- eBay Sandbox environment for API testing

## Project Structure

```
ebay-sniper/
├── frontend/                 # Next.js application
├── backend/                  # Python Lambda functions
├── infrastructure/           # AWS SAM/CloudFormation templates
├── docs/                     # Documentation
├── tests/                    # Test suites
└── scripts/                  # Deployment scripts
```

## Security Requirements

- All eBay tokens must be encrypted using AWS KMS
- Implement proper CORS policies
- Use HTTPS only
- Validate all user inputs
- Implement rate limiting per user
- Log all bid actions for audit trail

## eBay Compliance

- Maximum bid amount: $100,000
- Minimum bid timing: 5 seconds before auction end
- Respect eBay's rate limits (5000 API calls/day)
- Handle eBay API errors gracefully
- Implement proper retry logic with exponential backoff

## Testing Requirements

### Unit Tests
- Python backend: pytest
- Frontend: Jest + React Testing Library

### Integration Tests
- eBay API integration (using sandbox)
- DynamoDB operations
- Email sending functionality

### End-to-End Tests
- User registration and eBay linking
- Complete bid lifecycle
- Scheduled bid execution

## Deployment Pipeline

### Environments
- Development (dev)
- Staging (staging) 
- Production (prod)

### CI/CD
- GitHub Actions for automated deployment
- Separate AWS accounts per environment
- Infrastructure as Code using AWS SAM
- Automated testing before deployment

## Observability

### Logging
- CloudWatch Logs for all Lambda functions
- Structured JSON logging
- Include request IDs for traceability

### Metrics
- Custom CloudWatch metrics for bid success rates
- API Gateway metrics
- DynamoDB performance metrics

### Alerting
- Failed bid notifications
- API error rate thresholds
- High DynamoDB throttling alerts

## Error Handling

### eBay API Errors
- Handle rate limiting (HTTP 429)
- Retry with exponential backoff
- Graceful degradation for temporary outages

### Bid Execution Failures
- Email user about failed bids
- Log detailed error information
- Implement dead letter queues for failed jobs

## Coding Guidelines

- Always use descriptive variable names