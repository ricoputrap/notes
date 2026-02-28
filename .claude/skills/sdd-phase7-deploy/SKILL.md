---
name: sdd-phase7-deploy
description: Generate Dockerfile, deployment manifests, CI/CD pipeline, and deployment procedures
argument-hint: [project-details]
user-invocable: true
---

# Phase 7: Deployment Generation Skill

Generate production-ready deployment artifacts including Docker, Kubernetes/Docker Compose, CI/CD pipelines, and runbooks.

## How to Use This Skill

**Invoke manually:**
```
/sdd-phase7-deploy
/sdd-phase7-deploy NestJS PostgreSQL AWS
```

**What AI will do:**
1. Ask about deployment target (AWS, DigitalOcean, Azure, local, etc.)
2. Ask about orchestration (Kubernetes, Docker Compose, etc.)
3. Generate production Dockerfile (multi-stage build)
4. Generate deployment manifests (deployment.yml, service.yml)
5. Generate CI/CD pipeline (GitHub Actions, GitLab CI, etc.)
6. Generate pre-deployment checklist
7. Generate deployment and rollback procedures
8. Provide monitoring/alerting setup guidance

## Output Format

The generated deployment artifacts will include:

### 1. Dockerfile (Multi-Stage Build)
```dockerfile
# Stage 1: Build
FROM node:18-alpine AS builder

WORKDIR /build

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# Stage 2: Runtime
FROM node:18-alpine

WORKDIR /app

# Install dumb-init for proper signal handling
RUN apk add --no-cache dumb-init

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001

# Copy built application from builder
COPY --from=builder --chown=nodejs:nodejs /build/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /build/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /build/package*.json ./

USER nodejs

EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => {if (r.statusCode !== 200) throw new Error(r.statusCode)})"

ENTRYPOINT ["/usr/sbin/dumb-init", "--"]
CMD ["node", "dist/main.js"]
```

### 2. Docker Compose (Local Development)
```yaml
version: '3.9'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: development
      DATABASE_URL: postgresql://user:password@postgres:5432/task-api
      JWT_SECRET: ${JWT_SECRET}
    depends_on:
      postgres:
        condition: service_healthy
    volumes:
      - .:/app
      - /app/node_modules
    networks:
      - app-network

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: task-api
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  # Optional: Redis for caching
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    networks:
      - app-network

volumes:
  postgres_data:

networks:
  app-network:
    driver: bridge
```

### 3. Kubernetes Deployment Manifest
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: task-api
  namespace: default
  labels:
    app: task-api
    version: v1.0.0
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: task-api
  template:
    metadata:
      labels:
        app: task-api
        version: v1.0.0
    spec:
      containers:
      - name: task-api
        image: your-registry/task-api:v1.0.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3000
          name: http
          protocol: TCP
        env:
        - name: NODE_ENV
          value: "production"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: task-api-secrets
              key: database-url
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: task-api-secrets
              key: jwt-secret
        - name: LOG_LEVEL
          value: "info"

        # Resource limits
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"

        # Health checks
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3

        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3

        # Graceful shutdown
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]

      # Pod disruption budget (high availability)
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - task-api
              topologyKey: kubernetes.io/hostname

---
apiVersion: v1
kind: Service
metadata:
  name: task-api-service
  labels:
    app: task-api
spec:
  type: LoadBalancer
  selector:
    app: task-api
  ports:
  - port: 80
    targetPort: 3000
    protocol: TCP
    name: http
  sessionAffinity: None

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: task-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: task-api
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### 4. GitHub Actions CI/CD Pipeline
```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_pass
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
    - uses: actions/checkout@v3

    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'

    - name: Install dependencies
      run: npm ci

    - name: Run linter
      run: npm run lint

    - name: Run unit tests
      run: npm run test
      env:
        DATABASE_URL: postgresql://test_user:test_pass@localhost:5432/test_db

    - name: Run integration tests
      run: npm run test:e2e
      env:
        DATABASE_URL: postgresql://test_user:test_pass@localhost:5432/test_db

    - name: Upload coverage
      uses: codecov/codecov-action@v3
      with:
        files: ./coverage/lcov.info

  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write

    steps:
    - uses: actions/checkout@v3

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
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=semver,pattern={{version}}
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
    - uses: actions/checkout@v3

    - name: Deploy to production
      run: |
        # Configure kubectl
        mkdir -p $HOME/.kube
        echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > $HOME/.kube/config
        chmod 600 $HOME/.kube/config

        # Update deployment image
        kubectl set image deployment/task-api \
          task-api=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
          -n default

        # Wait for rollout
        kubectl rollout status deployment/task-api -n default

    - name: Verify deployment
      run: |
        kubectl get pods -l app=task-api
        kubectl logs -l app=task-api --tail=50

    - name: Health check
      run: |
        HEALTH_URL=$(kubectl get service task-api-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
        curl -f http://$HEALTH_URL/health || exit 1

    - name: Notify deployment
      if: success()
      uses: actions/github-script@v6
      with:
        script: |
          github.rest.issues.createComment({
            issue_number: context.issue.number,
            owner: context.repo.owner,
            repo: context.repo.repo,
            body: '✅ Deployed to production!'
          })
```

### 5. Pre-Deployment Checklist
```markdown
## Pre-Deployment Checklist

### Code Quality
- [ ] All tests pass locally: `npm run test`
- [ ] All E2E tests pass: `npm run test:e2e`
- [ ] No console.log statements in production code
- [ ] No hardcoded secrets or sensitive data
- [ ] Code reviewed and approved
- [ ] No linting errors: `npm run lint`

### Configuration
- [ ] Environment variables configured in deployment platform
- [ ] Database migrations prepared and tested
- [ ] Rollback plan documented
- [ ] Secrets securely stored (not in code)
- [ ] API keys and credentials rotated

### Security
- [ ] Input validation active on all endpoints
- [ ] HTTPS/TLS enforced
- [ ] CORS properly configured
- [ ] Rate limiting enabled
- [ ] Authentication and authorization working
- [ ] No sensitive data in logs
- [ ] Dependencies audited: `npm audit`

### Performance
- [ ] Database queries optimized
- [ ] Indexes created on frequently queried fields
- [ ] Slow query log reviewed
- [ ] Connection pooling configured
- [ ] Caching strategy in place (if needed)

### Monitoring & Observability
- [ ] Error tracking enabled (Sentry, DataDog, etc.)
- [ ] Performance monitoring configured
- [ ] Log aggregation set up
- [ ] Alerts configured for:
      - High error rate
      - High latency
      - Low uptime
      - High memory/CPU usage
- [ ] Dashboard created for key metrics
- [ ] Health check endpoint working

### Documentation
- [ ] README.md updated
- [ ] API documentation current (Swagger)
- [ ] Deployment guide written
- [ ] Runbook created (common tasks, troubleshooting)
- [ ] Incident response plan documented

### Infrastructure
- [ ] Database backups configured
- [ ] Backup restoration tested
- [ ] Load balancer configured (if needed)
- [ ] CDN configured (if needed)
- [ ] Autoscaling policies set
- [ ] Monitoring agents installed
- [ ] Log collection configured

### Testing the Deployment
- [ ] Health check returns 200 OK
- [ ] Can register a new user
- [ ] Can login with credentials
- [ ] API responds within SLA (200ms p95)
- [ ] Database is accessible and queryable
- [ ] External integrations working
- [ ] Email/notifications working

### Final Approval
- [ ] Product owner approves
- [ ] Tech lead approves
- [ ] DevOps/Infrastructure approves
- [ ] Stakeholders notified of deployment
```

### 6. Deployment Procedure
```bash
#!/bin/bash

set -e  # Exit on error

echo "🚀 Starting deployment..."

# 1. Run all tests
echo "✓ Running tests..."
npm run test
npm run test:e2e

# 2. Build Docker image
echo "✓ Building Docker image..."
TIMESTAMP=$(date +%s)
IMAGE_TAG="v1.0.0-$TIMESTAMP"
docker build -t task-api:$IMAGE_TAG .
docker tag task-api:$IMAGE_TAG task-api:latest

# 3. Push to registry
echo "✓ Pushing to registry..."
docker push task-api:$IMAGE_TAG
docker push task-api:latest

# 4. Run database migrations
echo "✓ Running database migrations..."
kubectl exec -it deployment/task-api -- npm run typeorm migration:run

# 5. Deploy to Kubernetes
echo "✓ Deploying to Kubernetes..."
kubectl set image deployment/task-api \
  task-api=task-api:$IMAGE_TAG \
  -n default

# 6. Wait for rollout
echo "✓ Waiting for rollout..."
kubectl rollout status deployment/task-api -n default --timeout=5m

# 7. Verify deployment
echo "✓ Verifying deployment..."
POD=$(kubectl get pods -l app=task-api -o jsonpath='{.items[0].metadata.name}')
kubectl logs $POD --tail=20

# 8. Health check
echo "✓ Health checking..."
HEALTH_URL=$(kubectl get service task-api-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
for i in {1..30}; do
  if curl -f http://$HEALTH_URL/health; then
    echo "✓ Health check passed!"
    break
  fi
  echo "Waiting for health check... ($i/30)"
  sleep 2
done

echo "✅ Deployment successful!"
echo "API URL: http://$HEALTH_URL"
```

### 7. Rollback Procedure
```bash
#!/bin/bash

echo "⚠️  Rolling back to previous version..."

# Find previous image tag
CURRENT_IMAGE=$(kubectl get deployment task-api -o jsonpath='{.spec.template.spec.containers[0].image}')
echo "Current image: $CURRENT_IMAGE"

# Option 1: Kubernetes built-in rollback
echo "Rolling back deployment..."
kubectl rollout undo deployment/task-api -n default

# Wait for rollout
kubectl rollout status deployment/task-api -n default

# Verify rollback
NEW_IMAGE=$(kubectl get deployment task-api -o jsonpath='{.spec.template.spec.containers[0].image}')
echo "New image: $NEW_IMAGE"

# Health check
HEALTH_URL=$(kubectl get service task-api-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -f http://$HEALTH_URL/health

echo "✅ Rollback complete!"
```

## Instructions for Claude

When the user invokes this skill:

1. **Ask About Deployment Target**
   - Cloud provider: AWS, Azure, Google Cloud, DigitalOcean, Heroku?
   - Self-hosted: Your own servers?
   - Local: Docker Compose for development?
   - Orchestration: Kubernetes, Docker Swarm, Nomad?

2. **Generate Dockerfile**
   - Multi-stage build (smaller final image)
   - Non-root user (security)
   - Health check endpoint
   - Signal handling (dumb-init or tini)
   - Efficient layer caching
   - Environment-specific configs (.dockerignore)

3. **Generate Deployment Configuration**
   - **Docker Compose:** Local development with PostgreSQL, Redis, etc.
   - **Kubernetes:** Production manifests (Deployment, Service, HPA, ConfigMap, Secret)
   - **Cloud-specific:** AWS CodeDeploy, Azure App Service, GCP Cloud Run, etc.

4. **Generate CI/CD Pipeline** (GitHub Actions example)
   - **Test stage:** Run tests against real database
   - **Build stage:** Build and push Docker image
   - **Deploy stage:** Update Kubernetes, run migrations, health check
   - **Rollback:** Automated or manual trigger
   - Coverage reports and artifact storage

5. **Generate Pre-Deployment Checklist**
   - Code quality (tests, linting, no secrets)
   - Configuration (env vars, migrations, secrets)
   - Security (input validation, HTTPS, CORS, rate limiting)
   - Performance (optimized queries, indexes)
   - Monitoring (error tracking, logs, alerts, dashboards)
   - Documentation (README, API docs, runbook)
   - Final approvals

6. **Generate Deployment Procedures**
   - Step-by-step deployment script
   - Database migration handling
   - Health checks and verification
   - Rollback procedures
   - Incident response

7. **Include Monitoring Setup Guidance**
   - Error tracking (Sentry, DataDog)
   - Performance monitoring (APM)
   - Log aggregation (ELK, CloudWatch)
   - Metrics collection (Prometheus, CloudWatch)
   - Alert thresholds

8. **Validate the Output**
   - Dockerfile is secure (non-root, no secrets)
   - Manifests are production-ready
   - CI/CD pipeline covers all phases (test, build, deploy)
   - Rollback strategy is clear
   - All checklists are comprehensive

9. **Provide Output** in organized sections:
   - **Section 1:** Dockerfile (copy to root)
   - **Section 2:** Deployment config (Kubernetes YAML or other)
   - **Section 3:** CI/CD pipeline (GitHub Actions or similar)
   - **Section 4:** Pre-deployment checklist (review before deploying)
   - **Section 5:** Deployment script (run for deployment)
   - **Section 6:** Rollback script (use if issues occur)
   - Each with clear instructions on where to place files

10. **Ask for Feedback**: "Review the pre-deployment checklist before running the deployment script. Rollback script is available if needed. Your system is now production-ready!"

## Example Interaction

```
User: /sdd-phase7-deploy

Claude: I'll generate complete deployment setup for your NestJS API.

Let me ask a few questions:
1. What's your deployment target? (AWS, Kubernetes, Docker Compose, etc.)
2. Should I generate Kubernetes manifests or Docker Compose?
3. Which CI/CD platform? (GitHub Actions, GitLab CI, etc.)
4. Do you need monitoring setup instructions?

[User answers]

Claude: Perfect! Here's your deployment package:

# Task Management API - Deployment Setup (Phase 7)

## 1. Dockerfile
[Multi-stage build optimized for production]

## 2. Kubernetes Manifests
[Deployment, Service, HPA with health checks]

## 3. GitHub Actions CI/CD
[Test → Build → Deploy → Verify pipeline]

## 4. Pre-Deployment Checklist
[Review before deploying]

## 5. Deployment Script
```bash
./deploy.sh
```

## 6. Rollback Script (If Needed)
```bash
./rollback.sh
```

**Next Steps:**
1. Review pre-deployment checklist
2. Configure secrets in your deployment platform
3. Run `./deploy.sh` to deploy to production
4. Monitor logs and metrics
5. Keep `./rollback.sh` handy just in case

You're ready to deploy! 🚀
```

## Tips for Best Results

- **Security first:** No secrets in code, use secret management
- **Health checks matter:** Make sure /health endpoint is robust
- **Graceful shutdown:** Give app time to close connections
- **Database migrations:** Always test in staging first
- **Rollback strategy:** Know how to rollback if something breaks
- **Monitoring before deploying:** Set up alerts before you need them
- **Documentation matters:** Future you will thank you
- **Test the rollback:** Don't learn how to rollback during an incident

## Related Skills

After completing deployment:
- Your SDD workflow is complete! You've gone from spec to production.
- Reference: [Spec-Driven Development Workflow](../../spec-driven-development-workflow.md) - Phase 7 details
- Next: Maintain and iterate based on production feedback

## Useful Tools

- **Docker Hub:** https://hub.docker.com
- **Kubernetes:** https://kubernetes.io/docs
- **Helm:** https://helm.sh (package manager for Kubernetes)
- **Terraform:** https://www.terraform.io (infrastructure as code)
- **GitHub Actions:** https://github.com/features/actions
