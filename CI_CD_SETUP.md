# GitHub Actions CI/CD Pipeline Configuration

This document contains the GitHub Actions workflow for automated testing and deployment.

Create this file at: `.github/workflows/deploy.yml`

```yaml
name: Deploy to AWS

on:
  push:
    branches:
      - main
      - develop
  pull_request:
    branches:
      - main

env:
  AWS_REGION: us-east-1
  REGISTRY: ghcr.io

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      mongodb:
        image: mongo:7.0
        options: >-
          --health-cmd mongosh
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 27017:27017
        env:
          MONGO_INITDB_ROOT_USERNAME: admin
          MONGO_INITDB_ROOT_PASSWORD: admin

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install pytest pytest-cov

      - name: Run tests
        env:
          MONGODB_URI: mongodb://admin:admin@localhost:27017/meetmind?authSource=admin
        run: |
          pytest --cov=. --cov-report=xml

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage.xml

  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name == 'push'

    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Log in to Container Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha

      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Deploy to Elastic Beanstalk
        run: |
          pip install awsebcli
          eb init -p docker meetmind-app --region ${{ env.AWS_REGION }} --quiet
          eb deploy meetmind-env

      - name: Run smoke tests
        run: |
          HEALTH_URL=$(aws elasticbeanstalk describe-environments \
            --environment-names meetmind-env \
            --query 'Environments[0].EndpointURL' \
            --output text)
          
          curl -f "http://${HEALTH_URL}/health" || exit 1

  notify:
    needs: [test, build, deploy]
    runs-on: ubuntu-latest
    if: always()

    steps:
      - name: Notify deployment status
        run: |
          if [ "${{ job.status }}" == "success" ]; then
            echo "✅ Deployment successful!"
          else
            echo "❌ Deployment failed!"
          fi
```

## Setup Instructions

### 1. Create the Workflow File
```bash
mkdir -p .github/workflows
# Copy the YAML content above to: .github/workflows/deploy.yml
git add .github/workflows/deploy.yml
git commit -m "Add GitHub Actions CI/CD pipeline"
git push
```

### 2. Configure GitHub Secrets
In GitHub repository settings, add these secrets:

- `AWS_ACCESS_KEY_ID`: Your AWS access key
- `AWS_SECRET_ACCESS_KEY`: Your AWS secret key

### 3. Pipeline Stages

**Stage 1: Test**
- Spin up MongoDB in Docker
- Run Python unit tests
- Generate coverage reports

**Stage 2: Build**
- Build Docker image
- Push to GitHub Container Registry
- Cache layers for faster rebuilds

**Stage 3: Deploy**
- Configure AWS credentials
- Deploy to Elastic Beanstalk
- Run smoke tests

### 4. Trigger Deployment
```bash
# Push to main branch to trigger pipeline
git push origin main
```

## GitHub Secrets Setup

1. Go to: **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret**
3. Add:
   - Name: `AWS_ACCESS_KEY_ID`
   - Value: Your AWS access key
4. Repeat for `AWS_SECRET_ACCESS_KEY`

## Pipeline Workflow

```
Code Push to main
    ↓
[TEST STAGE]
├─ Checkout code
├─ Setup Python 3.11
├─ Install dependencies
├─ Start MongoDB service
├─ Run pytest with coverage
└─ Upload coverage to Codecov
    ↓
[BUILD STAGE]
├─ Setup Docker Buildx
├─ Login to GitHub Container Registry
├─ Extract metadata (version, tags)
├─ Build Docker image
└─ Push to registry
    ↓
[DEPLOY STAGE]
├─ Configure AWS credentials
├─ Install Elastic Beanstalk CLI
├─ Deploy to meetmind-env
└─ Run health check
    ↓
[NOTIFY STAGE]
└─ Send deployment status
```

## Monitoring Deployments

### View Workflow Runs
1. Go to **Actions** tab
2. Click on workflow run
3. View logs for each job

### Check Deployment Status
```bash
# List recent deployments
aws elasticbeanstalk describe-events --environment-name meetmind-env

# Check environment health
aws elasticbeanstalk describe-environments --environment-names meetmind-env
```

## Troubleshooting

### GitHub Secrets Not Found
- ✅ Ensure secrets are created in repository settings
- ✅ Check secret names match exactly: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`

### Docker Push Fails
- ✅ GitHub token should have `write:packages` permission (default in Actions)
- ✅ Check GitHub Container Registry is public

### Beanstalk Deploy Fails
- ✅ Ensure `meetmind-env` exists
- ✅ Check AWS credentials have Beanstalk permissions
- ✅ Verify IAM role allows EC2 deployment

### Tests Fail
- ✅ MongoDB service must be healthy before running tests
- ✅ Check `MONGODB_URI` environment variable is set correctly

