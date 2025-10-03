# Templates de Configuración Docker

## Objetivo
Proporcionar templates estandarizados para la containerización de aplicaciones SAAS, incluyendo configuraciones para desarrollo, testing y producción.

## 1. Dockerfile para Aplicación Python/FastAPI

### Dockerfile Multi-stage para Producción
```dockerfile
# Multi-stage build for Python FastAPI application
FROM python:3.11-slim as builder

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

# Install system dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Create and activate virtual environment
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Copy requirements and install Python dependencies
COPY requirements.txt .
RUN pip install --upgrade pip && \
    pip install -r requirements.txt

# Production stage
FROM python:3.11-slim as production

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PATH="/opt/venv/bin:$PATH"

# Install runtime dependencies
RUN apt-get update && apt-get install -y \
    curl \
    && rm -rf /var/lib/apt/lists/* \
    && groupadd -r appuser && useradd -r -g appuser appuser

# Copy virtual environment from builder stage
COPY --from=builder /opt/venv /opt/venv

# Set working directory
WORKDIR /app

# Copy application code
COPY --chown=appuser:appuser . .

# Switch to non-root user
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Expose port
EXPOSE 8000

# Run application
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Dockerfile para Desarrollo
```dockerfile
# Development Dockerfile
FROM python:3.11-slim

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PYTHONPATH=/app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    curl \
    git \
    vim \
    && rm -rf /var/lib/apt/lists/*

# Set working directory
WORKDIR /app

# Install Python dependencies
COPY requirements.txt requirements-dev.txt ./
RUN pip install --upgrade pip && \
    pip install -r requirements.txt && \
    pip install -r requirements-dev.txt

# Copy application code
COPY . .

# Expose port
EXPOSE 8000

# Run application with hot reload
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
```

## 2. Dockerfile para Frontend (React/Next.js)

### Dockerfile Multi-stage para React
```dockerfile
# Multi-stage build for React application
FROM node:18-alpine as builder

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY . .

# Build application
RUN npm run build

# Production stage with Nginx
FROM nginx:alpine as production

# Copy custom nginx config
COPY nginx.conf /etc/nginx/nginx.conf

# Copy built application
COPY --from=builder /app/build /usr/share/nginx/html

# Add health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:80/ || exit 1

# Expose port
EXPOSE 80

# Start nginx
CMD ["nginx", "-g", "daemon off;"]
```

### Nginx Configuration
```nginx
# nginx.conf
events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    # Logging
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;

    # Performance
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml+rss
        application/atom+xml
        image/svg+xml;

    server {
        listen 80;
        server_name localhost;
        root /usr/share/nginx/html;
        index index.html index.htm;

        # Security headers
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header Referrer-Policy "no-referrer-when-downgrade" always;
        add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;

        # Handle client-side routing
        location / {
            try_files $uri $uri/ /index.html;
        }

        # Cache static assets
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }

        # Health check endpoint
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
    }
}
```

## 3. Docker Compose para Desarrollo

### docker-compose.yml
```yaml
version: '3.8'

services:
  # Backend API
  api:
    build:
      context: ./backend
      dockerfile: Dockerfile.dev
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://postgres:password@db:5432/saas_db
      - REDIS_URL=redis://redis:6379
      - ENVIRONMENT=development
    volumes:
      - ./backend:/app
      - /app/__pycache__
    depends_on:
      - db
      - redis
    networks:
      - saas-network

  # Frontend
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_API_URL=http://localhost:8000
    volumes:
      - ./frontend:/app
      - /app/node_modules
    networks:
      - saas-network

  # Database
  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=saas_db
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./database/init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - saas-network

  # Redis Cache
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - saas-network

  # Message Queue (RabbitMQ)
  rabbitmq:
    image: rabbitmq:3-management-alpine
    environment:
      - RABBITMQ_DEFAULT_USER=admin
      - RABBITMQ_DEFAULT_PASS=password
    ports:
      - "5672:5672"
      - "15672:15672"
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    networks:
      - saas-network

  # Monitoring - Prometheus
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
    networks:
      - saas-network

  # Monitoring - Grafana
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./monitoring/grafana/datasources:/etc/grafana/provisioning/datasources
    networks:
      - saas-network

volumes:
  postgres_data:
  redis_data:
  rabbitmq_data:
  prometheus_data:
  grafana_data:

networks:
  saas-network:
    driver: bridge
```

### docker-compose.override.yml (Para desarrollo local)
```yaml
version: '3.8'

services:
  api:
    environment:
      - DEBUG=true
      - LOG_LEVEL=debug
    volumes:
      - ./backend:/app
      - /app/__pycache__

  frontend:
    environment:
      - CHOKIDAR_USEPOLLING=true
    volumes:
      - ./frontend:/app
      - /app/node_modules

  db:
    environment:
      - POSTGRES_DB=saas_db_dev
    ports:
      - "5433:5432"
```

## 4. Docker Compose para Testing

### docker-compose.test.yml
```yaml
version: '3.8'

services:
  # Test Database
  test-db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=saas_test_db
      - POSTGRES_USER=test_user
      - POSTGRES_PASSWORD=test_password
    tmpfs:
      - /var/lib/postgresql/data
    networks:
      - test-network

  # Test Redis
  test-redis:
    image: redis:7-alpine
    tmpfs:
      - /data
    networks:
      - test-network

  # API Tests
  api-tests:
    build:
      context: ./backend
      dockerfile: Dockerfile.test
    environment:
      - DATABASE_URL=postgresql://test_user:test_password@test-db:5432/saas_test_db
      - REDIS_URL=redis://test-redis:6379
      - ENVIRONMENT=testing
    depends_on:
      - test-db
      - test-redis
    volumes:
      - ./backend:/app
      - ./test-results:/app/test-results
    command: ["pytest", "-v", "--cov=.", "--cov-report=xml", "--cov-report=html"]
    networks:
      - test-network

  # Frontend Tests
  frontend-tests:
    build:
      context: ./frontend
      dockerfile: Dockerfile.test
    volumes:
      - ./frontend:/app
      - ./test-results:/app/test-results
    command: ["npm", "test", "--", "--coverage", "--watchAll=false"]
    networks:
      - test-network

  # E2E Tests
  e2e-tests:
    build:
      context: ./e2e
      dockerfile: Dockerfile
    environment:
      - API_URL=http://api:8000
      - FRONTEND_URL=http://frontend:3000
    depends_on:
      - api
      - frontend
    volumes:
      - ./e2e:/app
      - ./test-results:/app/test-results
    networks:
      - test-network

networks:
  test-network:
    driver: bridge
```

## 5. Dockerfile para Testing

### Dockerfile.test (Backend)
```dockerfile
FROM python:3.11-slim

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PYTHONPATH=/app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Set working directory
WORKDIR /app

# Install Python dependencies
COPY requirements.txt requirements-test.txt ./
RUN pip install --upgrade pip && \
    pip install -r requirements.txt && \
    pip install -r requirements-test.txt

# Copy application code
COPY . .

# Create test results directory
RUN mkdir -p test-results

# Run tests by default
CMD ["pytest", "-v", "--cov=.", "--cov-report=xml:test-results/coverage.xml", "--cov-report=html:test-results/htmlcov"]
```

## 6. .dockerignore

### .dockerignore para Backend
```dockerignore
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
build/
develop-eggs/
dist/
downloads/
eggs/
.eggs/
lib/
lib64/
parts/
sdist/
var/
wheels/
*.egg-info/
.installed.cfg
*.egg

# Virtual environments
venv/
env/
ENV/

# IDE
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Git
.git/
.gitignore

# Docker
Dockerfile*
docker-compose*
.dockerignore

# Documentation
README.md
docs/

# Tests
tests/
test-results/
.coverage
htmlcov/

# Logs
*.log
logs/

# Environment files
.env*
!.env.example
```

### .dockerignore para Frontend
```dockerignore
# Dependencies
node_modules/
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# Production build
build/
dist/

# IDE
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Git
.git/
.gitignore

# Docker
Dockerfile*
docker-compose*
.dockerignore

# Documentation
README.md
docs/

# Tests
coverage/
.nyc_output/

# Environment files
.env*
!.env.example

# Logs
*.log
logs/
```

## 7. Scripts de Utilidad

### build.sh
```bash
#!/bin/bash
# Build script for Docker images

set -e

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Configuration
REGISTRY="your-registry.com"
PROJECT_NAME="saas-platform"
VERSION=${1:-latest}

echo -e "${GREEN}Building Docker images for ${PROJECT_NAME}:${VERSION}${NC}"

# Build backend
echo -e "${YELLOW}Building backend image...${NC}"
docker build -t ${REGISTRY}/${PROJECT_NAME}/api:${VERSION} ./backend

# Build frontend
echo -e "${YELLOW}Building frontend image...${NC}"
docker build -t ${REGISTRY}/${PROJECT_NAME}/frontend:${VERSION} ./frontend

# Build worker (if exists)
if [ -d "./worker" ]; then
    echo -e "${YELLOW}Building worker image...${NC}"
    docker build -t ${REGISTRY}/${PROJECT_NAME}/worker:${VERSION} ./worker
fi

echo -e "${GREEN}All images built successfully!${NC}"

# Tag as latest if version is not latest
if [ "$VERSION" != "latest" ]; then
    docker tag ${REGISTRY}/${PROJECT_NAME}/api:${VERSION} ${REGISTRY}/${PROJECT_NAME}/api:latest
    docker tag ${REGISTRY}/${PROJECT_NAME}/frontend:${VERSION} ${REGISTRY}/${PROJECT_NAME}/frontend:latest
    if [ -d "./worker" ]; then
        docker tag ${REGISTRY}/${PROJECT_NAME}/worker:${VERSION} ${REGISTRY}/${PROJECT_NAME}/worker:latest
    fi
fi

echo -e "${GREEN}Build completed successfully!${NC}"
```

### push.sh
```bash
#!/bin/bash
# Push script for Docker images

set -e

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Configuration
REGISTRY="your-registry.com"
PROJECT_NAME="saas-platform"
VERSION=${1:-latest}

echo -e "${GREEN}Pushing Docker images for ${PROJECT_NAME}:${VERSION}${NC}"

# Login to registry
echo -e "${YELLOW}Logging in to registry...${NC}"
docker login ${REGISTRY}

# Push images
echo -e "${YELLOW}Pushing backend image...${NC}"
docker push ${REGISTRY}/${PROJECT_NAME}/api:${VERSION}

echo -e "${YELLOW}Pushing frontend image...${NC}"
docker push ${REGISTRY}/${PROJECT_NAME}/frontend:${VERSION}

if [ -d "./worker" ]; then
    echo -e "${YELLOW}Pushing worker image...${NC}"
    docker push ${REGISTRY}/${PROJECT_NAME}/worker:${VERSION}
fi

# Push latest tags if version is not latest
if [ "$VERSION" != "latest" ]; then
    docker push ${REGISTRY}/${PROJECT_NAME}/api:latest
    docker push ${REGISTRY}/${PROJECT_NAME}/frontend:latest
    if [ -d "./worker" ]; then
        docker push ${REGISTRY}/${PROJECT_NAME}/worker:latest
    fi
fi

echo -e "${GREEN}All images pushed successfully!${NC}"
```

## Mejores Prácticas

### 1. Seguridad
- Usar imágenes base oficiales y actualizadas
- Ejecutar contenedores como usuario no-root
- Minimizar la superficie de ataque
- Escanear imágenes en busca de vulnerabilidades

### 2. Performance
- Usar multi-stage builds para reducir tamaño
- Optimizar el orden de las capas
- Usar .dockerignore para excluir archivos innecesarios
- Implementar health checks

### 3. Mantenibilidad
- Usar variables de entorno para configuración
- Documentar Dockerfiles claramente
- Versionar imágenes apropiadamente
- Implementar logging estructurado

### 4. Desarrollo
- Usar volúmenes para hot-reload en desarrollo
- Separar configuraciones de desarrollo y producción
- Implementar tests automatizados
- Usar docker-compose para orquestación local

Esta documentación proporciona templates completos y configuraciones optimizadas para containerizar aplicaciones SAAS de manera eficiente y segura.