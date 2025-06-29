# eBay Sniper

An automated bidding application for eBay auctions that allows users to set maximum bid amounts and automatically places bids 5 seconds before auction end.

## 🚀 Features

- **User Authentication**: Secure user registration and login via AWS Cognito
- **eBay Account Integration**: Link your eBay account to access wishlist items
- **Automated Bidding**: Set maximum bid amounts and let the system bid for you
- **Smart Timing**: Bids are placed 5 seconds before auction end for optimal timing
- **Wishlist Integration**: Select items from your eBay wishlist for bidding
- **Bid Management**: View, update, and cancel your active bids
- **Bid History**: Track your bidding history and auction outcomes
- **Email Notifications**: Receive notifications about bid results
- **Notification Preferences**: Customize your email notification settings

## 🏗️ Architecture

This is a full-stack serverless application built on AWS:

### Frontend
- **Framework**: Next.js 14+ with TypeScript
- **Styling**: Tailwind CSS
- **Deployment**: AWS Amplify
- **Package Manager**: pnpm

### Backend
- **Runtime**: Python 3.11+ on AWS Lambda
- **API Gateway**: HTTP API with Cognito authorization
- **Database**: Amazon DynamoDB
- **Scheduling**: AWS EventBridge for automated bid execution

### External Services
- **eBay API**: For auction data and bid placement
- **Postmark**: Email notifications
- **AWS Secrets Manager**: Secure credential storage

## 📋 Prerequisites

- **AWS Account** with appropriate permissions
- **eBay Developer Account** with API credentials
- **Postmark Account** for email notifications
- **Node.js** 18.x or later
- **Python** 3.11 or later
- **AWS CLI** v2 configured
- **AWS SAM CLI** for deployment

## 🛠️ Quick Start

### 1. Clone Repository
```bash
git clone https://github.com/your-org/ebay-sniper.git
cd ebay-sniper
```

### 2. Backend Setup
```bash
cd backend

# Create virtual environment
python3 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
pip install -r requirements-dev.txt

# Configure environment
cp .env.template .env.local
# Edit .env.local with your configuration
```

### 3. Frontend Setup
```bash
cd frontend

# Install dependencies
npm install -g pnpm
pnpm install

# Configure environment
cp .env.template .env.local
# Edit .env.local with your configuration
```

### 4. Deploy Infrastructure
```bash
# Build and deploy backend
cd backend
sam build
sam deploy --guided

# Deploy frontend
cd ../frontend
# Configure environment variables in AWS Amplify Console
git push origin main  # Triggers Amplify deployment
```

## 🔧 Development

### Local Development Setup

#### Start Backend (Terminal 1)
```bash
cd backend
source venv/bin/activate
uvicorn src.main:app --reload --host 0.0.0.0 --port 8000
```

#### Start Frontend (Terminal 2)
```bash
cd frontend
pnpm dev
```

#### Start DynamoDB Local (Terminal 3)
```bash
cd infrastructure/dynamodb
./start-dynamodb.sh
```

### Running Tests

#### Backend Tests
```bash
cd backend
pytest tests/ -v
pytest tests/ --cov=src --cov-report=html
```

#### Frontend Tests
```bash
cd frontend
pnpm test
pnpm test:e2e
```

## 📁 Project Structure

```
ebay-sniper/
├── frontend/                 # Next.js application
│   ├── pages/               # Next.js pages and API routes
│   ├── components/          # React components
│   ├── lib/                 # Utilities and API clients
│   ├── styles/              # Tailwind CSS styles
│   └── public/              # Static assets
├── backend/                  # Python Lambda functions
│   ├── src/
│   │   ├── main.py         # Lambda application entry point
│   │   ├── api/            # API route handlers
│   │   ├── models/         # Data models
│   │   ├── services/       # Business logic
│   │   └── utils/          # Helper utilities
│   ├── template.yaml       # SAM template
│   └── requirements.txt    # Python dependencies
├── infrastructure/           # AWS SAM/CloudFormation templates
├── docs/                     # Documentation
├── tests/                    # Test suites
└── scripts/                  # Deployment scripts
```

## 🔐 Security

- **Authentication**: AWS Cognito with MFA support
- **Authorization**: JWT tokens validated by API Gateway
- **Data Encryption**: Automatic encryption at rest via DynamoDB
- **Secrets Management**: AWS Secrets Manager for API credentials
- **Input Validation**: Comprehensive Pydantic model validation
- **Rate Limiting**: API Gateway and application-level throttling
- **HTTPS Only**: All communications encrypted in transit

## 🌍 Environment Configuration

### Required Environment Variables

#### Backend (.env.local)
```bash
# Development settings
DEBUG=true
DYNAMODB_ENDPOINT=http://localhost:8000

# External API keys (use AWS Secrets Manager in production)
POSTMARK_API_KEY=your_postmark_key
EBAY_APP_ID=your_ebay_app_id

# Optional - defaults provided
CORS_ORIGINS=http://localhost:3000
AWS_REGION=us-east-1
```

#### Frontend (.env.local)
```bash
# API Configuration
NEXT_PUBLIC_API_URL=http://localhost:8000

# AWS Configuration
NEXT_PUBLIC_AWS_REGION=us-east-1
NEXT_PUBLIC_USER_POOL_ID=us-east-1_xxxxxxxxx
NEXT_PUBLIC_USER_POOL_CLIENT_ID=your_client_id

# eBay Configuration
NEXT_PUBLIC_EBAY_ENVIRONMENT=sandbox
```

## 🚀 Deployment

### Development
```bash
sam deploy --config-env dev
```

### Staging
```bash
sam deploy --config-env staging
```

### Production
```bash
sam deploy --config-env prod
```

## 📚 Documentation

- [Architecture Overview](docs/architecture.md)
- [AWS Infrastructure](docs/aws-infrastructure.md)
- [Frontend Guide](docs/frontend.md)
- [Backend Guide](docs/backend.md)
- [Development Setup](docs/development-setup.md)
- [Deployment Guide](docs/deployment-guide.md)
- [Security Guide](docs/security-guide.md)
- [eBay API Integration](docs/ebay-api-integration.md)
- [Database Schema](docs/database-schema.md)

## 🔧 Commands Reference

### Backend Commands
```bash
# Development
uvicorn src.main:app --reload

# Testing
pytest tests/ -v

# Code formatting and linting
ruff format src/ tests/
ruff check src/ tests/

# Type checking
ty src/

# Deployment
sam build
sam deploy
```

### Frontend Commands
```bash
# Development
pnpm dev

# Testing and Quality
pnpm test
pnpm lint
pnpm type-check

# Build
pnpm build
```

## ⚖️ Legal Compliance

**Important**: This application involves automated bidding on eBay auctions. Usage must:

- ✅ Comply with eBay's Terms of Service and API usage policies
- ✅ Respect rate limiting and API guidelines (5000 calls/day)
- ✅ Consider ethical implications of automated bidding
- ✅ Ensure compliance with relevant laws and regulations
- ✅ Maximum bid limit: $100,000
- ✅ Minimum bid timing: 5 seconds before auction end

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

### Development Guidelines

- Follow existing code style and patterns
- Write comprehensive tests for new features
- Update documentation for API changes
- Ensure security best practices are followed
- Test thoroughly before submitting PRs

## 📄 License

This project is licensed under the GNU Affero General Public License v3 (AGPL-3.0) - see the [LICENSE](LICENSE) file for details.

## 🐛 Troubleshooting

### Common Issues

**Port conflicts**:
```bash
lsof -ti:8000 | xargs kill -9  # Backend
lsof -ti:3000 | xargs kill -9  # Frontend
```

**AWS credentials**:
```bash
aws configure list
aws sts get-caller-identity
```

**Python environment**:
```bash
rm -rf venv
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

**Node.js dependencies**:
```bash
pnpm store prune
rm -rf node_modules pnpm-lock.yaml
pnpm install
```

For more detailed troubleshooting, see the [Development Setup Guide](docs/development-setup.md).

## 📞 Support

For questions, issues, or contributions:

- Create an [Issue](https://github.com/your-org/ebay-sniper/issues)
- Check the [Documentation](docs/)
- Review the [Development Setup Guide](docs/development-setup.md)

---

**Disclaimer**: This application is for educational and personal use. Users are responsible for ensuring compliance with eBay's terms of service and applicable laws.
