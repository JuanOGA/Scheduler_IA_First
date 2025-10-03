# Templates de CI/CD

## Objetivo
Proporcionar templates estandarizados para pipelines de CI/CD, incluyendo configuraciones para GitHub Actions, GitLab CI, y Jenkins, con soporte para testing, building, security scanning y deployment.

## 1. GitHub Actions Templates

### .github/workflows/ci.yml
```yaml
name: CI Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # Code Quality and Security
  code-quality:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      with:
        fetch-depth: 0

    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'

    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install flake8 black isort mypy bandit safety
        if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
        if [ -f requirements-dev.txt ]; then pip install -r requirements-dev.txt; fi

    - name: Lint with flake8
      run: |
        flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
        flake8 . --count --exit-zero --max-complexity=10 --max-line-length=127 --statistics

    - name: Check code formatting with black
      run: black --check .

    - name: Check import sorting with isort
      run: isort --check-only .

    - name: Type checking with mypy
      run: mypy .

    - name: Security check with bandit
      run: bandit -r . -f json -o bandit-report.json

    - name: Check dependencies with safety
      run: safety check --json --output safety-report.json

    - name: Upload security reports
      uses: actions/upload-artifact@v3
      if: always()
      with:
        name: security-reports
        path: |
          bandit-report.json
          safety-report.json

  # Backend Tests
  backend-tests:
    runs-on: ubuntu-latest
    needs: code-quality
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'

    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        pip install -r requirements-test.txt

    - name: Run tests with pytest
      env:
        DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db
        REDIS_URL: redis://localhost:6379
        ENVIRONMENT: testing
      run: |
        pytest --cov=. --cov-report=xml --cov-report=html --junitxml=pytest-report.xml

    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml
        flags: backend
        name: backend-coverage

    - name: Upload test results
      uses: actions/upload-artifact@v3
      if: always()
      with:
        name: test-results
        path: |
          pytest-report.xml
          htmlcov/

  # Frontend Tests
  frontend-tests:
    runs-on: ubuntu-latest
    needs: code-quality

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '18'
        cache: 'npm'
        cache-dependency-path: frontend/package-lock.json

    - name: Install dependencies
      working-directory: ./frontend
      run: npm ci

    - name: Run ESLint
      working-directory: ./frontend
      run: npm run lint

    - name: Run tests
      working-directory: ./frontend
      run: npm test -- --coverage --watchAll=false

    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./frontend/coverage/lcov.info
        flags: frontend
        name: frontend-coverage

  # Build and Push Images
  build-and-push:
    runs-on: ubuntu-latest
    needs: [backend-tests, frontend-tests]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    permissions:
      contents: read
      packages: write

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Log in to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=sha,prefix={{branch}}-
          type=raw,value=latest,enable={{is_default_branch}}

    # Build Backend
    - name: Build and push backend image
      uses: docker/build-push-action@v5
      with:
        context: ./backend
        push: true
        tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/api:${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

    # Build Frontend
    - name: Build and push frontend image
      uses: docker/build-push-action@v5
      with:
        context: ./frontend
        push: true
        tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/frontend:${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

  # Security Scanning
  security-scan:
    runs-on: ubuntu-latest
    needs: build-and-push
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    steps:
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/api:latest
        format: 'sarif'
        output: 'trivy-results.sarif'

    - name: Upload Trivy scan results to GitHub Security tab
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'
```

### .github/workflows/cd.yml
```yaml
name: CD Pipeline

on:
  workflow_run:
    workflows: ["CI Pipeline"]
    types:
      - completed
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    environment: staging

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-west-2

    - name: Set up kubectl
      uses: azure/setup-kubectl@v3
      with:
        version: 'v1.28.0'

    - name: Update kubeconfig
      run: |
        aws eks update-kubeconfig --region us-west-2 --name staging-cluster

    - name: Deploy to staging
      run: |
        # Update image tags in Kubernetes manifests
        sed -i "s|image: .*api:.*|image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/api:${{ github.sha }}|g" k8s/staging/api-deployment.yaml
        sed -i "s|image: .*frontend:.*|image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/frontend:${{ github.sha }}|g" k8s/staging/frontend-deployment.yaml
        
        # Apply manifests
        kubectl apply -f k8s/staging/
        
        # Wait for rollout
        kubectl rollout status deployment/api -n saas-platform-staging
        kubectl rollout status deployment/frontend -n saas-platform-staging

    - name: Run smoke tests
      run: |
        # Wait for services to be ready
        sleep 30
        
        # Run basic health checks
        curl -f https://staging-api.saas-platform.com/health
        curl -f https://staging.saas-platform.com/

  deploy-production:
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment: production
    if: github.ref == 'refs/heads/main'

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-west-2

    - name: Set up kubectl
      uses: azure/setup-kubectl@v3
      with:
        version: 'v1.28.0'

    - name: Update kubeconfig
      run: |
        aws eks update-kubeconfig --region us-west-2 --name production-cluster

    - name: Deploy to production
      run: |
        # Update image tags in Kubernetes manifests
        sed -i "s|image: .*api:.*|image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/api:${{ github.sha }}|g" k8s/production/api-deployment.yaml
        sed -i "s|image: .*frontend:.*|image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/frontend:${{ github.sha }}|g" k8s/production/frontend-deployment.yaml
        
        # Apply manifests with rolling update
        kubectl apply -f k8s/production/
        
        # Wait for rollout
        kubectl rollout status deployment/api -n saas-platform
        kubectl rollout status deployment/frontend -n saas-platform

    - name: Run production health checks
      run: |
        # Wait for services to be ready
        sleep 60
        
        # Run comprehensive health checks
        curl -f https://api.saas-platform.com/health
        curl -f https://saas-platform.com/
        
        # Check metrics endpoint
        curl -f https://api.saas-platform.com/metrics

    - name: Notify deployment success
      uses: 8398a7/action-slack@v3
      with:
        status: success
        text: 'Production deployment successful! :rocket:'
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

## 2. GitLab CI Templates

### .gitlab-ci.yml
```yaml
stages:
  - validate
  - test
  - build
  - security
  - deploy-staging
  - deploy-production

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"
  REGISTRY: $CI_REGISTRY
  IMAGE_TAG: $CI_COMMIT_SHA
  POSTGRES_DB: test_db
  POSTGRES_USER: postgres
  POSTGRES_PASSWORD: postgres
  REDIS_URL: redis://redis:6379

# Templates
.docker_template: &docker_template
  image: docker:24.0.5
  services:
    - docker:24.0.5-dind
  before_script:
    - echo $CI_REGISTRY_PASSWORD | docker login -u $CI_REGISTRY_USER --password-stdin $CI_REGISTRY

.kubectl_template: &kubectl_template
  image: bitnami/kubectl:latest
  before_script:
    - kubectl config use-context $KUBE_CONTEXT

# Validation Stage
code-quality:
  stage: validate
  image: python:3.11
  before_script:
    - pip install flake8 black isort mypy bandit safety
    - pip install -r requirements.txt
    - pip install -r requirements-dev.txt
  script:
    - flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
    - black --check .
    - isort --check-only .
    - mypy .
    - bandit -r . -f json -o bandit-report.json
    - safety check --json --output safety-report.json
  artifacts:
    reports:
      junit: bandit-report.json
    paths:
      - bandit-report.json
      - safety-report.json
    expire_in: 1 week
  only:
    - merge_requests
    - main
    - develop

# Test Stage
backend-tests:
  stage: test
  image: python:3.11
  services:
    - postgres:15
    - redis:7
  variables:
    DATABASE_URL: postgresql://postgres:postgres@postgres:5432/test_db
  before_script:
    - pip install -r requirements.txt
    - pip install -r requirements-test.txt
  script:
    - pytest --cov=. --cov-report=xml --cov-report=html --junitxml=pytest-report.xml
  coverage: '/TOTAL.+ ([0-9]{1,3}%)/'
  artifacts:
    reports:
      junit: pytest-report.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
    paths:
      - htmlcov/
    expire_in: 1 week
  only:
    - merge_requests
    - main
    - develop

frontend-tests:
  stage: test
  image: node:18
  before_script:
    - cd frontend
    - npm ci
  script:
    - npm run lint
    - npm test -- --coverage --watchAll=false
  coverage: '/Lines\s*:\s*(\d+\.\d+)%/'
  artifacts:
    reports:
      junit: frontend/junit.xml
      coverage_report:
        coverage_format: cobertura
        path: frontend/coverage/cobertura-coverage.xml
    paths:
      - frontend/coverage/
    expire_in: 1 week
  only:
    - merge_requests
    - main
    - develop

# Build Stage
build-backend:
  stage: build
  <<: *docker_template
  script:
    - docker build -t $REGISTRY/$CI_PROJECT_PATH/api:$IMAGE_TAG ./backend
    - docker push $REGISTRY/$CI_PROJECT_PATH/api:$IMAGE_TAG
    - docker tag $REGISTRY/$CI_PROJECT_PATH/api:$IMAGE_TAG $REGISTRY/$CI_PROJECT_PATH/api:latest
    - docker push $REGISTRY/$CI_PROJECT_PATH/api:latest
  only:
    - main

build-frontend:
  stage: build
  <<: *docker_template
  script:
    - docker build -t $REGISTRY/$CI_PROJECT_PATH/frontend:$IMAGE_TAG ./frontend
    - docker push $REGISTRY/$CI_PROJECT_PATH/frontend:$IMAGE_TAG
    - docker tag $REGISTRY/$CI_PROJECT_PATH/frontend:$IMAGE_TAG $REGISTRY/$CI_PROJECT_PATH/frontend:latest
    - docker push $REGISTRY/$CI_PROJECT_PATH/frontend:latest
  only:
    - main

# Security Stage
container-security-scan:
  stage: security
  image: aquasec/trivy:latest
  script:
    - trivy image --format template --template "@contrib/sarif.tpl" -o trivy-report.sarif $REGISTRY/$CI_PROJECT_PATH/api:$IMAGE_TAG
    - trivy image --format template --template "@contrib/sarif.tpl" -o trivy-frontend-report.sarif $REGISTRY/$CI_PROJECT_PATH/frontend:$IMAGE_TAG
  artifacts:
    reports:
      sast: 
        - trivy-report.sarif
        - trivy-frontend-report.sarif
    expire_in: 1 week
  only:
    - main

# Deploy Staging
deploy-staging:
  stage: deploy-staging
  <<: *kubectl_template
  environment:
    name: staging
    url: https://staging.saas-platform.com
  script:
    - |
      # Update image tags
      sed -i "s|image: .*api:.*|image: $REGISTRY/$CI_PROJECT_PATH/api:$IMAGE_TAG|g" k8s/staging/api-deployment.yaml
      sed -i "s|image: .*frontend:.*|image: $REGISTRY/$CI_PROJECT_PATH/frontend:$IMAGE_TAG|g" k8s/staging/frontend-deployment.yaml
      
      # Apply manifests
      kubectl apply -f k8s/staging/
      
      # Wait for rollout
      kubectl rollout status deployment/api -n saas-platform-staging --timeout=300s
      kubectl rollout status deployment/frontend -n saas-platform-staging --timeout=300s
      
      # Health check
      sleep 30
      curl -f https://staging-api.saas-platform.com/health
  only:
    - main

# Deploy Production
deploy-production:
  stage: deploy-production
  <<: *kubectl_template
  environment:
    name: production
    url: https://saas-platform.com
  when: manual
  script:
    - |
      # Update image tags
      sed -i "s|image: .*api:.*|image: $REGISTRY/$CI_PROJECT_PATH/api:$IMAGE_TAG|g" k8s/production/api-deployment.yaml
      sed -i "s|image: .*frontend:.*|image: $REGISTRY/$CI_PROJECT_PATH/frontend:$IMAGE_TAG|g" k8s/production/frontend-deployment.yaml
      
      # Apply manifests
      kubectl apply -f k8s/production/
      
      # Wait for rollout
      kubectl rollout status deployment/api -n saas-platform --timeout=600s
      kubectl rollout status deployment/frontend -n saas-platform --timeout=600s
      
      # Health check
      sleep 60
      curl -f https://api.saas-platform.com/health
      curl -f https://saas-platform.com/
  only:
    - main
```

## 3. Jenkins Pipeline

### Jenkinsfile
```groovy
pipeline {
    agent any
    
    environment {
        REGISTRY = 'your-registry.com'
        IMAGE_NAME = 'saas-platform'
        DOCKER_CREDENTIALS = credentials('docker-registry-credentials')
        KUBECONFIG = credentials('kubeconfig')
        SLACK_WEBHOOK = credentials('slack-webhook-url')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()
                    env.BUILD_VERSION = "${env.BUILD_NUMBER}-${env.GIT_COMMIT_SHORT}"
                }
            }
        }
        
        stage('Code Quality') {
            parallel {
                stage('Backend Linting') {
                    steps {
                        script {
                            docker.image('python:3.11').inside {
                                sh '''
                                    pip install flake8 black isort mypy bandit safety
                                    pip install -r requirements.txt
                                    pip install -r requirements-dev.txt
                                    
                                    flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
                                    black --check .
                                    isort --check-only .
                                    mypy .
                                    bandit -r . -f json -o bandit-report.json
                                    safety check --json --output safety-report.json
                                '''
                            }
                        }
                        publishHTML([
                            allowMissing: false,
                            alwaysLinkToLastBuild: true,
                            keepAll: true,
                            reportDir: '.',
                            reportFiles: 'bandit-report.json',
                            reportName: 'Bandit Security Report'
                        ])
                    }
                }
                
                stage('Frontend Linting') {
                    steps {
                        dir('frontend') {
                            script {
                                docker.image('node:18').inside {
                                    sh '''
                                        npm ci
                                        npm run lint
                                    '''
                                }
                            }
                        }
                    }
                }
            }
        }
        
        stage('Tests') {
            parallel {
                stage('Backend Tests') {
                    steps {
                        script {
                            docker.image('python:3.11').inside('--link postgres:postgres --link redis:redis') {
                                sh '''
                                    pip install -r requirements.txt
                                    pip install -r requirements-test.txt
                                    
                                    export DATABASE_URL=postgresql://postgres:postgres@postgres:5432/test_db
                                    export REDIS_URL=redis://redis:6379
                                    export ENVIRONMENT=testing
                                    
                                    pytest --cov=. --cov-report=xml --cov-report=html --junitxml=pytest-report.xml
                                '''
                            }
                        }
                        publishTestResults testResultsPattern: 'pytest-report.xml'
                        publishCoverage adapters: [
                            coberturaAdapter('coverage.xml')
                        ], sourceFileResolver: sourceFiles('STORE_LAST_BUILD')
                    }
                }
                
                stage('Frontend Tests') {
                    steps {
                        dir('frontend') {
                            script {
                                docker.image('node:18').inside {
                                    sh '''
                                        npm ci
                                        npm test -- --coverage --watchAll=false
                                    '''
                                }
                            }
                        }
                        publishTestResults testResultsPattern: 'frontend/junit.xml'
                    }
                }
            }
        }
        
        stage('Build Images') {
            when {
                branch 'main'
            }
            parallel {
                stage('Build Backend') {
                    steps {
                        script {
                            def backendImage = docker.build("${REGISTRY}/${IMAGE_NAME}/api:${BUILD_VERSION}", "./backend")
                            docker.withRegistry("https://${REGISTRY}", 'docker-registry-credentials') {
                                backendImage.push()
                                backendImage.push("latest")
                            }
                        }
                    }
                }
                
                stage('Build Frontend') {
                    steps {
                        script {
                            def frontendImage = docker.build("${REGISTRY}/${IMAGE_NAME}/frontend:${BUILD_VERSION}", "./frontend")
                            docker.withRegistry("https://${REGISTRY}", 'docker-registry-credentials') {
                                frontendImage.push()
                                frontendImage.push("latest")
                            }
                        }
                    }
                }
            }
        }
        
        stage('Security Scan') {
            when {
                branch 'main'
            }
            steps {
                script {
                    sh """
                        docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \\
                            -v \$(pwd):/tmp/trivy aquasec/trivy:latest image \\
                            --format template --template "@contrib/sarif.tpl" \\
                            -o /tmp/trivy/trivy-report.sarif \\
                            ${REGISTRY}/${IMAGE_NAME}/api:${BUILD_VERSION}
                    """
                }
                publishHTML([
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: '.',
                    reportFiles: 'trivy-report.sarif',
                    reportName: 'Trivy Security Report'
                ])
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            steps {
                script {
                    sh """
                        # Update image tags
                        sed -i "s|image: .*api:.*|image: ${REGISTRY}/${IMAGE_NAME}/api:${BUILD_VERSION}|g" k8s/staging/api-deployment.yaml
                        sed -i "s|image: .*frontend:.*|image: ${REGISTRY}/${IMAGE_NAME}/frontend:${BUILD_VERSION}|g" k8s/staging/frontend-deployment.yaml
                        
                        # Apply manifests
                        kubectl apply -f k8s/staging/
                        
                        # Wait for rollout
                        kubectl rollout status deployment/api -n saas-platform-staging --timeout=300s
                        kubectl rollout status deployment/frontend -n saas-platform-staging --timeout=300s
                    """
                }
                
                // Health check
                script {
                    sleep(30)
                    sh 'curl -f https://staging-api.saas-platform.com/health'
                    sh 'curl -f https://staging.saas-platform.com/'
                }
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            input {
                message "Deploy to production?"
                ok "Deploy"
                parameters {
                    choice(
                        name: 'DEPLOYMENT_STRATEGY',
                        choices: ['rolling', 'blue-green'],
                        description: 'Deployment strategy'
                    )
                }
            }
            steps {
                script {
                    sh """
                        # Update image tags
                        sed -i "s|image: .*api:.*|image: ${REGISTRY}/${IMAGE_NAME}/api:${BUILD_VERSION}|g" k8s/production/api-deployment.yaml
                        sed -i "s|image: .*frontend:.*|image: ${REGISTRY}/${IMAGE_NAME}/frontend:${BUILD_VERSION}|g" k8s/production/frontend-deployment.yaml
                        
                        # Apply manifests
                        kubectl apply -f k8s/production/
                        
                        # Wait for rollout
                        kubectl rollout status deployment/api -n saas-platform --timeout=600s
                        kubectl rollout status deployment/frontend -n saas-platform --timeout=600s
                    """
                }
                
                // Health check
                script {
                    sleep(60)
                    sh 'curl -f https://api.saas-platform.com/health'
                    sh 'curl -f https://saas-platform.com/'
                    sh 'curl -f https://api.saas-platform.com/metrics'
                }
            }
        }
    }
    
    post {
        always {
            // Clean up
            sh 'docker system prune -f'
            
            // Archive artifacts
            archiveArtifacts artifacts: '**/*.xml,**/*.json,**/*.sarif', allowEmptyArchive: true
        }
        
        success {
            script {
                if (env.BRANCH_NAME == 'main') {
                    slackSend(
                        channel: '#deployments',
                        color: 'good',
                        message: ":rocket: Production deployment successful!\nVersion: ${BUILD_VERSION}\nCommit: ${GIT_COMMIT_SHORT}"
                    )
                }
            }
        }
        
        failure {
            slackSend(
                channel: '#deployments',
                color: 'danger',
                message: ":x: Pipeline failed!\nBranch: ${env.BRANCH_NAME}\nBuild: ${env.BUILD_NUMBER}"
            )
        }
    }
}
```

## 4. Terraform CI/CD

### .github/workflows/terraform.yml
```yaml
name: Terraform CI/CD

on:
  push:
    branches: [ main ]
    paths: [ 'infrastructure/**' ]
  pull_request:
    branches: [ main ]
    paths: [ 'infrastructure/**' ]

env:
  TF_VERSION: '1.6.0'
  AWS_REGION: 'us-west-2'

jobs:
  terraform-validate:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: ${{ env.TF_VERSION }}

    - name: Terraform Format Check
      working-directory: ./infrastructure
      run: terraform fmt -check -recursive

    - name: Terraform Init
      working-directory: ./infrastructure
      run: terraform init -backend=false

    - name: Terraform Validate
      working-directory: ./infrastructure
      run: terraform validate

    - name: Run tfsec
      uses: aquasecurity/tfsec-action@v1.0.3
      with:
        working_directory: ./infrastructure

  terraform-plan:
    runs-on: ubuntu-latest
    needs: terraform-validate
    if: github.event_name == 'pull_request'
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: ${{ env.TF_VERSION }}

    - name: Terraform Init
      working-directory: ./infrastructure
      run: terraform init

    - name: Terraform Plan
      working-directory: ./infrastructure
      run: |
        terraform plan -out=tfplan -no-color
        terraform show -no-color tfplan > tfplan.txt

    - name: Comment PR with plan
      uses: actions/github-script@v7
      with:
        script: |
          const fs = require('fs');
          const plan = fs.readFileSync('./infrastructure/tfplan.txt', 'utf8');
          const maxGitHubBodyCharacters = 65536;
          
          function chunkSubstr(str, size) {
            const numChunks = Math.ceil(str.length / size)
            const chunks = new Array(numChunks)
            for (let i = 0, o = 0; i < numChunks; ++i, o += size) {
              chunks[i] = str.substr(o, size)
            }
            return chunks
          }
          
          const body = plan.length > maxGitHubBodyCharacters ? 
            `\`\`\`\n${plan.substring(0, maxGitHubBodyCharacters)}\n\`\`\`\n\n*Plan truncated*` : 
            `\`\`\`\n${plan}\n\`\`\``;
          
          github.rest.issues.createComment({
            issue_number: context.issue.number,
            owner: context.repo.owner,
            repo: context.repo.repo,
            body: `## Terraform Plan\n\n${body}`
          });

  terraform-apply:
    runs-on: ubuntu-latest
    needs: terraform-validate
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: ${{ env.TF_VERSION }}

    - name: Terraform Init
      working-directory: ./infrastructure
      run: terraform init

    - name: Terraform Apply
      working-directory: ./infrastructure
      run: terraform apply -auto-approve
```

## 5. Monitoring y Alertas

### .github/workflows/monitoring.yml
```yaml
name: Monitoring and Alerts

on:
  schedule:
    - cron: '*/5 * * * *'  # Every 5 minutes
  workflow_dispatch:

jobs:
  health-check:
    runs-on: ubuntu-latest
    steps:
    - name: Check API Health
      run: |
        response=$(curl -s -o /dev/null -w "%{http_code}" https://api.saas-platform.com/health)
        if [ $response -ne 200 ]; then
          echo "API health check failed with status: $response"
          exit 1
        fi

    - name: Check Frontend Health
      run: |
        response=$(curl -s -o /dev/null -w "%{http_code}" https://saas-platform.com/)
        if [ $response -ne 200 ]; then
          echo "Frontend health check failed with status: $response"
          exit 1
        fi

    - name: Notify on failure
      if: failure()
      uses: 8398a7/action-slack@v3
      with:
        status: failure
        text: 'Health check failed! :warning:'
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

## Mejores Prácticas

### 1. Seguridad
- Usar secrets para credenciales sensibles
- Escanear imágenes por vulnerabilidades
- Implementar análisis de código estático
- Validar dependencias

### 2. Testing
- Ejecutar tests en paralelo
- Mantener cobertura de código alta
- Implementar tests de integración
- Usar ambientes de testing aislados

### 3. Deployment
- Implementar rolling deployments
- Usar health checks
- Mantener rollback automático
- Monitorear post-deployment

### 4. Monitoreo
- Implementar alertas proactivas
- Monitorear métricas clave
- Usar distributed tracing
- Mantener logs centralizados

Esta documentación proporciona templates completos para implementar pipelines de CI/CD robustos y automatizados para aplicaciones SAAS.