# Part 17: Docker Integration

## Module 2: Intermediate Level (ระดับกลาง)

---

## ขั้นตอนที่ 381: Docker Basics

```bash
#!/usr/bin/env bash
# docker_basics.sh - Docker Basics

echo "=== Docker Integration ==="
echo ""

# ตรวจสอบ Docker
if ! command -v docker &>/dev/null; then
    echo "Docker ไม่ได้ติดตั้ง"
    echo "ติดตั้งได้ที่: https://docs.docker.com/get-docker/"
    echo ""
    echo "Docker commands หลัก (reference):"
    cat << 'EOF'
# Images
docker pull image:tag          # download image
docker images                  # list images
docker rmi image:tag           # ลบ image
docker build -t name:tag .     # build image
docker push registry/name:tag  # push to registry

# Containers
docker run [options] image     # รัน container
docker ps                      # list running
docker ps -a                   # list all
docker stop container          # stop gracefully
docker kill container          # force stop
docker rm container            # ลบ container
docker rm $(docker ps -aq)     # ลบทั้งหมด

# Exec & Logs
docker exec -it container bash  # shell เข้า container
docker logs container           # ดู logs
docker logs -f container        # follow logs
docker inspect container        # ดู details

# Volumes
docker volume create myvolume
docker volume ls
docker volume rm myvolume

# Networks
docker network create mynet
docker network ls
docker network inspect mynet
EOF
    exit 0
fi

echo "Docker version:"
docker version --format "Client: {{.Client.Version}}, Server: {{.Server.Version}}" 2>/dev/null || \
docker version | head -5

echo ""
echo "Docker info:"
docker info 2>/dev/null | grep -E "Containers|Images|Server Version" | head -5
```

---

## ขั้นตอนที่ 382: Docker Container Management

```bash
#!/usr/bin/env bash
# docker_containers.sh - Container Management

echo "=== Docker Container Management ==="

# ==================== Container Functions ====================

# รัน container
run_container() {
    local name="$1"
    local image="$2"
    shift 2
    local extra_args=("$@")
    
    echo "Starting container: $name"
    docker run -d \
        --name "$name" \
        --restart unless-stopped \
        "${extra_args[@]}" \
        "$image"
    
    echo "Container started: $(docker inspect --format='{{.Id}}' "$name" 2>/dev/null | head -c 12)"
}

# หยุด container
stop_container() {
    local name="$1"
    local timeout="${2:-10}"
    
    if docker ps -q -f name="$name" | grep -q .; then
        echo "Stopping $name..."
        docker stop -t "$timeout" "$name"
        echo "Stopped: $name"
    else
        echo "$name is not running"
    fi
}

# ลบ container
remove_container() {
    local name="$1"
    
    stop_container "$name"
    
    if docker ps -aq -f name="$name" | grep -q .; then
        docker rm "$name"
        echo "Removed: $name"
    fi
}

# ตรวจสอบ status
container_status() {
    local name="$1"
    
    if docker ps -q -f name="$name" | grep -q .; then
        echo "RUNNING"
    elif docker ps -aq -f name="$name" | grep -q .; then
        echo "STOPPED"
    else
        echo "NOT FOUND"
    fi
}

# ดู logs
container_logs() {
    local name="$1"
    local lines="${2:-50}"
    local follow="${3:-false}"
    
    local args=("--tail" "$lines")
    if [[ "$follow" == "true" ]]; then
        args+=("-f")
    fi
    
    docker logs "${args[@]}" "$name" 2>/dev/null
}

echo "Container management functions available"
echo ""

# Demo ถ้ามี Docker
if command -v docker &>/dev/null && docker info &>/dev/null 2>&1; then
    echo "Docker is available. Running demo..."
    echo ""
    
    echo "1. รัน Nginx:"
    run_container "test-nginx" "nginx:alpine" \
        -p 8090:80 \
        -v /tmp/nginx_html:/usr/share/nginx/html 2>/dev/null || \
        echo "  (ต้องมี Docker daemon running)"
    
    sleep 2
    
    echo ""
    echo "2. Status:"
    echo "  nginx status: $(container_status "test-nginx")"
    
    echo ""
    echo "3. Logs:"
    container_logs "test-nginx" 5 2>/dev/null || true
    
    echo ""
    echo "4. Cleanup:"
    remove_container "test-nginx" 2>/dev/null || true
else
    echo "Docker daemon ไม่พร้อมใช้งาน"
fi
```

---

## ขั้นตอนที่ 383: Dockerfile และ Image Building

```bash
#!/usr/bin/env bash
# dockerfile_build.sh - Build Docker Images

echo "=== Dockerfile และ Image Building ==="

echo "1. Basic Dockerfile:"
cat << 'DOCKERFILE'
# Dockerfile
FROM ubuntu:22.04

# Metadata
LABEL maintainer="yourname@example.com"
LABEL version="1.0"

# Build arguments
ARG APP_VERSION=1.0.0
ARG BUILD_DATE

# Environment variables
ENV APP_HOME=/app \
    APP_PORT=8080 \
    APP_VERSION=${APP_VERSION}

# Install dependencies
RUN apt-get update && apt-get install -y \
    curl \
    jq \
    && rm -rf /var/lib/apt/lists/*

# Create app directory
WORKDIR $APP_HOME

# Copy files
COPY scripts/ ./scripts/
COPY config/ ./config/
COPY entrypoint.sh .

# Set permissions
RUN chmod +x entrypoint.sh scripts/*.sh

# Create non-root user
RUN useradd -r -s /bin/false appuser && \
    chown -R appuser:appuser $APP_HOME
USER appuser

# Expose port
EXPOSE $APP_PORT

# Health check
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD curl -f http://localhost:$APP_PORT/health || exit 1

# Entrypoint
ENTRYPOINT ["./entrypoint.sh"]
CMD ["start"]
DOCKERFILE

echo ""
echo "2. Multi-stage build:"
cat << 'DOCKERFILE'
# Build stage
FROM node:18-alpine AS builder

WORKDIR /build
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Final stage (smaller image)
FROM node:18-alpine AS runtime

WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY --from=builder --chown=appuser:appgroup /build/dist ./dist
COPY --from=builder --chown=appuser:appgroup /build/node_modules ./node_modules

USER appuser
EXPOSE 3000
CMD ["node", "dist/index.js"]
DOCKERFILE

echo ""
echo "3. Build script:"
cat << 'SCRIPT'
#!/usr/bin/env bash
# build.sh

set -euo pipefail

IMAGE_NAME="${IMAGE_NAME:-myapp}"
IMAGE_TAG="${IMAGE_TAG:-$(git rev-parse --short HEAD 2>/dev/null || echo 'latest')}"
REGISTRY="${REGISTRY:-}"
BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)

FULL_IMAGE="${REGISTRY}${IMAGE_NAME}:${IMAGE_TAG}"

echo "Building: $FULL_IMAGE"

docker build \
    --build-arg BUILD_DATE="$BUILD_DATE" \
    --build-arg APP_VERSION="$IMAGE_TAG" \
    --label "build.date=$BUILD_DATE" \
    --label "build.version=$IMAGE_TAG" \
    -t "$FULL_IMAGE" \
    -t "${REGISTRY}${IMAGE_NAME}:latest" \
    .

echo "✓ Build successful: $FULL_IMAGE"

# Push ถ้าระบุ registry
if [[ -n "$REGISTRY" ]]; then
    echo "Pushing to registry..."
    docker push "$FULL_IMAGE"
    docker push "${REGISTRY}${IMAGE_NAME}:latest"
    echo "✓ Pushed: $FULL_IMAGE"
fi
SCRIPT
```

---

## ขั้นตอนที่ 384: Docker Compose

```bash
#!/usr/bin/env bash
# docker_compose.sh - Docker Compose

echo "=== Docker Compose ==="

echo "1. docker-compose.yml example:"
cat << 'YAML'
# docker-compose.yml
version: '3.8'

services:
  # Web application
  app:
    build: .
    image: myapp:latest
    container_name: myapp
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - NODE_ENV=production
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=myapp
      - DB_USER=myapp
      - DB_PASS=${DB_PASSWORD}  # จาก .env file
      - REDIS_URL=redis://redis:6379
    volumes:
      - ./logs:/app/logs
      - app_uploads:/app/uploads
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - backend
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
  
  # PostgreSQL database
  postgres:
    image: postgres:15-alpine
    container_name: postgres
    restart: unless-stopped
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=myapp
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapp"]
      interval: 10s
      timeout: 5s
      retries: 5
  
  # Redis cache
  redis:
    image: redis:7-alpine
    container_name: redis
    restart: unless-stopped
    volumes:
      - redis_data:/data
    networks:
      - backend
  
  # Nginx reverse proxy
  nginx:
    image: nginx:alpine
    container_name: nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - app
    networks:
      - backend

volumes:
  postgres_data:
  redis_data:
  app_uploads:

networks:
  backend:
    driver: bridge
YAML

echo ""
echo "2. Compose management script:"
cat << 'SCRIPT'
#!/usr/bin/env bash
# compose_manager.sh

COMPOSE_FILE="${COMPOSE_FILE:-docker-compose.yml}"
ENV_FILE="${ENV_FILE:-.env}"

compose() {
    docker-compose -f "$COMPOSE_FILE" --env-file "$ENV_FILE" "$@"
}

case "${1:-help}" in
    up)
        echo "Starting services..."
        compose up -d "${@:2}"
        compose ps
        ;;
    down)
        echo "Stopping services..."
        compose down "${@:2}"
        ;;
    restart)
        compose restart "${@:2}"
        ;;
    logs)
        compose logs --tail=100 -f "${@:2}"
        ;;
    status)
        compose ps
        ;;
    build)
        compose build --no-cache "${@:2}"
        ;;
    exec)
        compose exec "${@:2}"
        ;;
    *)
        echo "Usage: $0 {up|down|restart|logs|status|build|exec}"
        ;;
esac
SCRIPT
```

---

## ขั้นตอนที่ 385: Docker Cleanup และ Maintenance

```bash
#!/usr/bin/env bash
# docker_cleanup.sh - Docker Cleanup

echo "=== Docker Cleanup ==="

echo "1. ดู Docker disk usage:"
docker system df 2>/dev/null || echo "(Docker ไม่พร้อม)"

echo ""
echo "2. Cleanup functions:"
cat << 'FUNCTIONS'
#!/usr/bin/env bash

# ลบ stopped containers
cleanup_containers() {
    echo "Removing stopped containers..."
    docker container prune -f
    echo "Done"
}

# ลบ dangling images (unused layers)
cleanup_images() {
    echo "Removing dangling images..."
    docker image prune -f
    
    # ลบ images เก่า (older than 7 days)
    # docker image prune -a --filter "until=168h" -f
    echo "Done"
}

# ลบ unused volumes
cleanup_volumes() {
    echo "Removing unused volumes..."
    docker volume prune -f
    echo "Done"
}

# ลบ unused networks
cleanup_networks() {
    echo "Removing unused networks..."
    docker network prune -f
    echo "Done"
}

# Full cleanup
cleanup_all() {
    echo "=== Full Docker Cleanup ==="
    
    local before_size
    before_size=$(docker system df 2>/dev/null | awk 'NR>1 {sum+=$4} END {print sum}' || echo 0)
    
    cleanup_containers
    cleanup_images
    cleanup_volumes
    cleanup_networks
    
    local after_size
    after_size=$(docker system df 2>/dev/null | awk 'NR>1 {sum+=$4} END {print sum}' || echo 0)
    
    echo ""
    echo "Space freed: $(( before_size - after_size )) MB (approx)"
    
    # Or use docker system prune
    # docker system prune -af --volumes
}

# Scheduled cleanup
schedule_cleanup() {
    # เพิ่มใน crontab: 0 2 * * 0 /usr/local/bin/docker_cleanup.sh
    echo "Add to crontab for weekly cleanup:"
    echo "  0 2 * * 0 docker system prune -af --volumes >> /var/log/docker_cleanup.log 2>&1"
}
FUNCTIONS

echo ""
echo "3. Resource limits:"
cat << 'LIMITS'
# รัน container พร้อม resource limits
docker run -d \
    --name myapp \
    --memory="512m" \
    --memory-swap="1g" \
    --cpus="0.5" \
    --cpu-shares=512 \
    --pids-limit=100 \
    --ulimit nofile=1024:1024 \
    myapp:latest
LIMITS

echo ""
echo "4. Monitor resource usage:"
# docker stats ไม่ block ด้วย --no-stream
docker stats --no-stream 2>/dev/null | head -5 || echo "(Docker ไม่พร้อม)"
```

---

## ขั้นตอนที่ 386: Workshop - CI/CD Pipeline Script

```bash
#!/usr/bin/env bash
# cicd_pipeline.sh - Workshop: CI/CD Pipeline

set -euo pipefail

# ==================== CI/CD Pipeline ====================

echo "=== CI/CD Pipeline Workshop ==="

# Configuration
PROJECT_NAME="${PROJECT_NAME:-myapp}"
BUILD_NUMBER="${BUILD_NUMBER:-local}"
GIT_BRANCH=$(git rev-parse --abbrev-ref HEAD 2>/dev/null || echo "unknown")
GIT_COMMIT=$(git rev-parse --short HEAD 2>/dev/null || echo "unknown")
REGISTRY="${REGISTRY:-docker.io/myorg}"

IMAGE_TAG="${GIT_BRANCH//\//-}-${GIT_COMMIT}"
FULL_IMAGE="${REGISTRY}/${PROJECT_NAME}:${IMAGE_TAG}"

# Colors
GREEN='\033[0;32m'; RED='\033[0;31m'; YELLOW='\033[1;33m'; NC='\033[0m'

# ==================== Pipeline Stages ====================

stage() {
    local name="$1"
    echo ""
    echo -e "${YELLOW}╔══════════════════════════════════════╗${NC}"
    echo -e "${YELLOW}║ Stage: ${name}${NC}"
    echo -e "${YELLOW}╚══════════════════════════════════════╝${NC}"
}

success() { echo -e "  ${GREEN}✓ $*${NC}"; }
fail()    { echo -e "  ${RED}✗ $*${NC}"; }

# Stage 1: Test
run_tests() {
    stage "TEST"
    echo "Running tests..."
    
    # Unit tests
    if [[ -f "package.json" ]]; then
        npm test 2>/dev/null && success "Node.js tests" || fail "Node.js tests"
    fi
    
    if [[ -f "pytest.ini" ]] || [[ -f "setup.py" ]]; then
        python -m pytest 2>/dev/null && success "Python tests" || fail "Python tests"
    fi
    
    # Shell script tests
    if command -v bats &>/dev/null; then
        bats tests/ 2>/dev/null && success "Shell tests" || fail "Shell tests"
    fi
    
    success "Tests passed (simulated)"
}

# Stage 2: Lint
run_lint() {
    stage "LINT"
    echo "Running linters..."
    
    local failed=0
    
    # ShellCheck
    if command -v shellcheck &>/dev/null; then
        find . -name "*.sh" -not -path "./.git/*" -exec shellcheck {} \; 2>/dev/null && \
            success "ShellCheck" || { fail "ShellCheck"; failed=1; }
    fi
    
    # Dockerfile lint
    if command -v hadolint &>/dev/null && [[ -f "Dockerfile" ]]; then
        hadolint Dockerfile 2>/dev/null && \
            success "Dockerfile lint" || { fail "Dockerfile lint"; failed=1; }
    fi
    
    if (( failed == 0 )); then
        success "Linting passed"
    fi
}

# Stage 3: Build
run_build() {
    stage "BUILD"
    echo "Building image: $FULL_IMAGE"
    
    if ! command -v docker &>/dev/null || ! docker info &>/dev/null 2>&1; then
        echo "  (Docker ไม่พร้อม, simulating...)"
        success "Build simulated: $FULL_IMAGE"
        return
    fi
    
    if [[ -f "Dockerfile" ]]; then
        docker build \
            --label "git.commit=$GIT_COMMIT" \
            --label "git.branch=$GIT_BRANCH" \
            --label "build.number=$BUILD_NUMBER" \
            -t "$FULL_IMAGE" \
            . && success "Image built: $FULL_IMAGE" || {
            fail "Build failed"
            return 1
        }
    else
        success "Build (no Dockerfile, skipped)"
    fi
}

# Stage 4: Security Scan
run_security_scan() {
    stage "SECURITY"
    echo "Running security checks..."
    
    # Check for secrets
    if git ls-files 2>/dev/null | xargs grep -l -E '(password|secret|api_key|token)\s*=' 2>/dev/null; then
        fail "Possible secrets found!"
    else
        success "No secrets detected"
    fi
    
    # Container scan
    if command -v trivy &>/dev/null; then
        trivy image "$FULL_IMAGE" 2>/dev/null && \
            success "Container scan passed" || fail "Security vulnerabilities found"
    else
        success "Security scan (skipped, trivy not installed)"
    fi
}

# Stage 5: Deploy
run_deploy() {
    local environment="${1:-staging}"
    stage "DEPLOY: $environment"
    
    echo "Deploying to $environment..."
    echo "  Image: $FULL_IMAGE"
    echo "  Branch: $GIT_BRANCH"
    echo "  Commit: $GIT_COMMIT"
    
    # Production deploy safety check
    if [[ "$environment" == "production" ]] && [[ "$GIT_BRANCH" != "main" ]]; then
        fail "Production deploy ต้องมาจาก main branch เท่านั้น!"
        return 1
    fi
    
    success "Deployment successful (simulated)"
}

# ==================== Main Pipeline ====================
main() {
    echo "Pipeline: ${PROJECT_NAME} #${BUILD_NUMBER}"
    echo "Branch: ${GIT_BRANCH}, Commit: ${GIT_COMMIT}"
    echo ""
    
    local start_time=$SECONDS
    
    run_tests
    run_lint
    run_build
    run_security_scan
    
    # Deploy ตาม branch
    case "$GIT_BRANCH" in
        main|master)
            run_deploy "production"
            ;;
        develop|staging)
            run_deploy "staging"
            ;;
        feature/*)
            run_deploy "dev"
            ;;
        *)
            echo ""
            echo "Branch '$GIT_BRANCH': no deployment configured"
            ;;
    esac
    
    local elapsed=$(( SECONDS - start_time ))
    echo ""
    echo -e "${GREEN}✓ Pipeline completed in ${elapsed}s${NC}"
}

main "$@"
```

---

## ขั้นตอนที่ 387: สรุป Part 17 - Docker

```bash
#!/usr/bin/env bash
# summary_docker.sh

echo "=== สรุป Docker Integration ==="
echo ""
echo "Docker Concepts:"
echo "  Image   - template (immutable)"
echo "  Container - running instance"
echo "  Volume  - persistent storage"
echo "  Network - container communication"
echo "  Registry - image storage"
echo ""
echo "Key Commands:"
echo "  docker run, stop, rm"
echo "  docker build, push, pull"
echo "  docker exec, logs, inspect"
echo "  docker ps, images"
echo "  docker-compose up/down"
echo ""
echo "Best Practices:"
echo "  - ใช้ non-root user"
echo "  - Multi-stage builds"
echo "  - .dockerignore"
echo "  - Resource limits"
echo "  - Health checks"
echo "  - Immutable containers"
echo ""
echo "Next: Part 18 - System Administration"
```

---

## สรุป

| หัวข้อ | Steps |
|--------|-------|
| Docker basics | 381 |
| Container management | 382 |
| Dockerfile | 383 |
| Docker Compose | 384 |
| Cleanup/maintenance | 385 |
| Workshop: CI/CD | 386 |

**ขั้นตอนต่อไป**: Part 18 - System Administration Scripts
