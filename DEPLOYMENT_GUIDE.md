# Deployment Guide - Notification Service

## Pre-Deployment Checklist

### System Requirements
- [ ] Docker installed (version 20.10+)
- [ ] Docker Compose installed (version 1.29+)
- [ ] At least 2GB RAM available
- [ ] Ports 8080, 5432, 5672, 15672 available
- [ ] At least 2GB disk space

### Verification
```bash
docker --version
docker-compose --version
docker run hello-world
```

---

## Local Development Deployment

### Option 1: Full Docker Deployment (Recommended)

#### Step 1: Clone/Navigate to Project
```bash
cd notification-service
```

#### Step 2: Create Environment File
```bash
cp .env.example .env
```

**Optional**: Edit .env to change configuration
```
API_PORT=8080              # API listening port
NODE_ENV=production        # Environment
LOG_LEVEL=info            # Logging level
JWT_SECRET=your_secret_key_change_in_production
```

#### Step 3: Build and Start
```bash
docker-compose up --build
```

**Expected Output**:
```
Creating notification-db       ... done
Creating notification-rabbitmq ... done
Creating notification-api      ... done
Creating notification-worker   ... done
```

#### Step 4: Verify Services
```bash
# Check health
curl http://localhost:8080/health

# Check readiness
curl http://localhost:8080/ready

# View logs
docker-compose logs -f
```

#### Step 5: Test API
```bash
curl -X POST http://localhost:8080/api/notifications \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiI5ZmM4MWZkZC1mYzljLTQ5ZTMtOWRmMS1hMzZkZjdlMWVjNWEiLCJ1c2VybmFtZSI6InRlc3R1c2VyIiwiaWF0IjoxNzAzNjAwMDAwfQ.GyTdZVu2VUh8UZ5qmyY8xR2x8xN9pQ6rT5xK3jL4z8M" \
  -d '{
    "targetUserId": "9fc81fdd-fc9c-49e3-9df1-a36df7e1ec5a",
    "type": "in-app",
    "payload": {"title": "Test", "message": "Hello"}
  }'
```

---

### Option 2: Local Development (Without Docker)

#### Prerequisites
- Node.js 18+
- PostgreSQL 15
- RabbitMQ 3.12
- npm or yarn

#### Step 1: Install Dependencies
```bash
cd api && npm install && cd ..
cd worker && npm install && cd ..
```

#### Step 2: Start Database Services
```bash
# Start PostgreSQL
docker run -d --name postgres \
  -e POSTGRES_DB=notification_db \
  -e POSTGRES_USER=notification_user \
  -e POSTGRES_PASSWORD=notification_password \
  -p 5432:5432 \
  postgres:15-alpine

# Start RabbitMQ
docker run -d --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  rabbitmq:3.12-management-alpine
```

#### Step 3: Initialize Database
```bash
psql -h localhost -U notification_user -d notification_db -f db/init.sql
```

#### Step 4: Create .env File
```bash
cp .env.example .env
```

#### Step 5: Start Services
```bash
# Terminal 1: API Service
cd api
npm start

# Terminal 2: Worker Service
cd worker
npm start
```

---

## Production Deployment

### Pre-Production Checklist
- [ ] Change JWT_SECRET to secure random value
- [ ] Set NODE_ENV=production
- [ ] Configure LOG_LEVEL=warn or error
- [ ] Set up database backups
- [ ] Configure log aggregation service
- [ ] Set up monitoring and alerting
- [ ] Review security settings
- [ ] Test failover procedures

### Docker Compose Production Configuration

Create `docker-compose.prod.yml`:

```yaml
version: '3.8'

services:
  db:
    image: postgres:15-alpine
    container_name: notification-db-prod
    environment:
      POSTGRES_DB: notification_db
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    ports:
      - "5432:5432"
    volumes:
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql
      - postgres_data_prod:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d notification_db"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: always
    networks:
      - notification_network_prod

  rabbitmq:
    image: rabbitmq:3.12-management-alpine
    container_name: notification-rabbitmq-prod
    ports:
      - "5672:5672"
      - "15672:15672"
    volumes:
      - rabbitmq_data_prod:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: always
    networks:
      - notification_network_prod

  api:
    build: ./api
    container_name: notification-api-prod
    ports:
      - "${API_PORT}:${API_PORT}"
    environment:
      NODE_ENV: production
      API_PORT: ${API_PORT}
      DATABASE_URL: postgres://${DB_USER}:${DB_PASSWORD}@db:5432/notification_db
      RABBITMQ_URL: amqp://guest:guest@rabbitmq:5672
      JWT_SECRET: ${JWT_SECRET}
      LOG_LEVEL: warn
    depends_on:
      db:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:${API_PORT}/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
      - notification_network_prod
    restart: always
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 256M
        reservations:
          cpus: '0.5'
          memory: 128M

  worker:
    build: ./worker
    container_name: notification-worker-prod
    environment:
      NODE_ENV: production
      DATABASE_URL: postgres://${DB_USER}:${DB_PASSWORD}@db:5432/notification_db
      RABBITMQ_URL: amqp://guest:guest@rabbitmq:5672
      QUEUE_NAME: notifications_queue
      DLQ_NAME: notifications_dlq
      MAX_RETRIES: 3
      LOG_LEVEL: warn
    depends_on:
      db:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    networks:
      - notification_network_prod
    restart: always
    deploy:
      replicas: 3  # Multiple workers for load distribution
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
        reservations:
          cpus: '0.25'
          memory: 128M

volumes:
  postgres_data_prod:
  rabbitmq_data_prod:

networks:
  notification_network_prod:
    driver: bridge
```

### Production Environment Variables (.env.prod)

```bash
# API
API_PORT=8080
NODE_ENV=production
LOG_LEVEL=warn

# Database
DB_USER=notification_user
DB_PASSWORD=$(openssl rand -base64 32)  # Generate secure password

# Database URL (full)
DATABASE_URL=postgres://notification_user:${DB_PASSWORD}@db:5432/notification_db

# RabbitMQ
RABBITMQ_URL=amqp://guest:guest@rabbitmq:5672

# Security
JWT_SECRET=$(openssl rand -base64 64)  # Generate secure secret

# Worker
QUEUE_NAME=notifications_queue
DLQ_NAME=notifications_dlq
MAX_RETRIES=3
```

### Production Deployment Steps

#### Step 1: Generate Secure Secrets
```bash
# Generate secure passwords
openssl rand -base64 32  # Database password
openssl rand -base64 64  # JWT secret
```

#### Step 2: Create Production Environment
```bash
cat > .env.prod << EOF
API_PORT=8080
NODE_ENV=production
LOG_LEVEL=warn

DATABASE_URL=postgres://notification_user:YOUR_DB_PASSWORD@db:5432/notification_db
RABBITMQ_URL=amqp://guest:guest@rabbitmq:5672

JWT_SECRET=YOUR_JWT_SECRET

QUEUE_NAME=notifications_queue
DLQ_NAME=notifications_dlq
MAX_RETRIES=3
EOF
```

#### Step 3: Deploy
```bash
# Build with production config
docker-compose -f docker-compose.prod.yml up --build -d

# Verify
docker-compose -f docker-compose.prod.yml ps
docker-compose -f docker-compose.prod.yml logs -f api
```

#### Step 4: Health Check
```bash
curl http://localhost:8080/health

# Should return:
# {
#   "status": "healthy",
#   "database": "connected",
#   "rabbitmq": "connected"
# }
```

---

## Kubernetes Deployment

### Create Kubernetes Manifests

#### deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notification-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: notification-api
  template:
    metadata:
      labels:
        app: notification-api
    spec:
      containers:
      - name: api
        image: notification-api:latest
        ports:
        - containerPort: 8080
        env:
        - name: NODE_ENV
          value: production
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: notification-secrets
              key: database-url
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: notification-secrets
              key: jwt-secret
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        resources:
          requests:
            memory: "128Mi"
            cpu: "250m"
          limits:
            memory: "256Mi"
            cpu: "500m"

---
apiVersion: v1
kind: Service
metadata:
  name: notification-api-service
spec:
  type: LoadBalancer
  selector:
    app: notification-api
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```

#### Deploy to Kubernetes
```bash
# Create namespace
kubectl create namespace notification-service

# Create secrets
kubectl create secret generic notification-secrets \
  --from-literal=database-url="postgres://..." \
  --from-literal=jwt-secret="..." \
  -n notification-service

# Deploy
kubectl apply -f deployment.yaml -n notification-service

# Verify
kubectl get pods -n notification-service
kubectl logs -f deployment/notification-api -n notification-service
```

---

## Monitoring & Logging

### Set Up Log Aggregation

#### ELK Stack Example
```bash
# Container names and log drivers
docker-compose logs --follow api | tee api.log
```

#### CloudWatch Integration
```bash
# In docker-compose.yml, add:
logging:
  driver: awslogs
  options:
    awslogs-group: notification-service
    awslogs-region: us-east-1
```

### Set Up Monitoring

#### Prometheus Metrics
Add to API:
```javascript
const prometheus = require('prom-client');

const httpRequestDuration = new prometheus.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code']
});
```

#### Alert Rules
```yaml
alert: NotificationServiceDown
  expr: up{job="notification-api"} == 0
  for: 5m

alert: HighErrorRate
  expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
  for: 5m

alert: QueueBacklog
  expr: rabbitmq_queue_messages_ready > 1000
  for: 10m
```

---

## Maintenance

### Database Backup
```bash
# Daily backup
0 2 * * * docker-compose exec db pg_dump -U notification_user notification_db > backup_$(date +\%Y\%m\%d).sql

# Restore from backup
docker-compose exec db psql -U notification_user notification_db < backup_20240121.sql
```

### Log Rotation
```bash
# In docker-compose.yml:
logging:
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"
```

### Update Services
```bash
# Pull latest images
docker-compose pull

# Rebuild
docker-compose build --no-cache

# Restart
docker-compose up -d
```

---

## Troubleshooting Deployment

### Services Won't Start
```bash
# Check logs
docker-compose logs api
docker-compose logs worker
docker-compose logs db

# Restart services
docker-compose restart

# Clean and rebuild
docker-compose down -v
docker-compose up --build
```

### Port Already in Use
```bash
# Find and kill process
lsof -ti:8080 | xargs kill -9

# Or change port in .env
API_PORT=8081
docker-compose up --build
```

### Database Connection Failed
```bash
# Check database is healthy
docker-compose ps db

# Check logs
docker-compose logs db

# Verify credentials
docker-compose exec db psql -U notification_user -d notification_db -c "SELECT NOW();"
```

### RabbitMQ Queue Issues
```bash
# Access management UI
# http://localhost:15672

# Check queue status
curl http://localhost:15672/api/queues

# Purge queue if needed
curl -i -u guest:guest -XDELETE http://localhost:15672/api/queues/%2F/notifications_queue
```

---

## Performance Optimization

### Database
```sql
-- Analyze query performance
EXPLAIN ANALYZE 
  SELECT * FROM notifications 
  WHERE user_id = 'uuid' 
  AND created_at > NOW() - INTERVAL '1 day';

-- Maintenance
VACUUM notifications;
ANALYZE notifications;
```

### RabbitMQ
```bash
# Increase memory limit
docker-compose exec rabbitmq rabbitmqctl set_vm_memory_high_watermark 0.6

# Monitor memory usage
docker-compose exec rabbitmq rabbitmqctl status
```

### Application
- Increase worker replicas for throughput
- Tune connection pool sizes
- Enable compression for large payloads
- Implement caching for frequently accessed data

---

## Security Hardening

### Firewall Rules
```bash
# Allow only necessary ports
ufw allow 22/tcp    # SSH
ufw allow 8080/tcp  # API
ufw allow 5672/tcp  # RabbitMQ (internal only)
```

### Database Security
```sql
-- Limit connections
ALTER USER notification_user WITH CONNECTION LIMIT 50;

-- Require strong passwords
ALTER USER notification_user WITH PASSWORD 'secure_random_password';
```

### API Security
```javascript
// Add rate limiting
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100 // limit each IP to 100 requests per windowMs
});

app.use('/api/', limiter);
```

---

## Rollback Procedure

### If Deployment Fails
```bash
# Stop current version
docker-compose down

# Restore from backup
git checkout previous-version

# Restart with previous version
docker-compose up --build

# Verify health
curl http://localhost:8080/health
```

---

## Deployment Checklist - Final

- [ ] System requirements verified
- [ ] Docker and Docker Compose installed
- [ ] Ports 8080, 5432, 5672, 15672 available
- [ ] Environment variables configured
- [ ] JWT_SECRET changed from default
- [ ] Database backups configured
- [ ] Log aggregation set up
- [ ] Monitoring and alerts configured
- [ ] Health checks passing
- [ ] API endpoints tested
- [ ] Database persistence verified
- [ ] Worker processing messages
- [ ] Documentation available
- [ ] Rollback plan in place

---

## Support & Next Steps

1. **Read Documentation**: Start with README.md
2. **Run Tests**: Execute `node test-client.js`
3. **Monitor**: Check `docker-compose logs -f`
4. **Scale**: Add worker instances as needed
5. **Optimize**: Fine-tune based on metrics

---

**Deployment Complete!** Your notification service is ready for use. 🚀
