# 🚀 Production Deployment Guide

Complete guide for deploying the Product Management SaaS to production environments.

## Pre-deployment Checklist

### Security
- [ ] Change all default passwords
- [ ] Update JWT secrets with strong random values
- [ ] Enable HTTPS/SSL certificates
- [ ] Configure firewall rules
- [ ] Set up WAF (Web Application Firewall)
- [ ] Enable two-factor authentication
- [ ] Review and update CORS settings
- [ ] Set secure headers (HSTS, CSP, X-Frame-Options)

### Performance
- [ ] Enable database connection pooling
- [ ] Configure CDN for static assets
- [ ] Enable gzip compression
- [ ] Set up caching strategy with Redis
- [ ] Configure image optimization

### Infrastructure
- [ ] Set up database backups
- [ ] Configure monitoring and alerting
- [ ] Set up log aggregation
- [ ] Configure auto-scaling
- [ ] Test failover procedures

## AWS Deployment

### Architecture Overview

```
CloudFront (CDN)
     ↓
Application Load Balancer (ALB)
     ↓
Auto Scaling Group (ECS/Fargate)
     ├→ Backend Service
     └→ Frontend Service
     ↓
RDS Database (PostgreSQL)
ElastiCache (Redis)
S3 (File Storage)
```

### Steps

1. **Create ECR Repositories**
```bash
aws ecr create-repository --repository-name product-mgmt-backend --region us-east-1
aws ecr create-repository --repository-name product-mgmt-frontend --region us-east-1
```

2. **Build and Push Docker Images**
```bash
# Login to ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com

# Backend
cd backend
docker build -t product-mgmt-backend:latest .
docker tag product-mgmt-backend:latest <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/product-mgmt-backend:latest
docker push <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/product-mgmt-backend:latest

# Frontend
cd ../frontend
docker build -t product-mgmt-frontend:latest .
docker tag product-mgmt-frontend:latest <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/product-mgmt-frontend:latest
docker push <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/product-mgmt-frontend:latest
```

3. **Create RDS Database**
```bash
aws rds create-db-instance \
  --db-instance-identifier product-mgmt-prod \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --engine-version 15.3 \
  --master-username postgres \
  --master-user-password "StrongPassword123!" \
  --allocated-storage 100 \
  --storage-encrypted \
  --multi-az \
  --publicly-accessible false
```

4. **Create ElastiCache Redis**
```bash
aws elasticache create-cache-cluster \
  --cache-cluster-id product-mgmt-redis \
  --cache-node-type cache.t3.micro \
  --engine redis \
  --engine-version 7.0 \
  --num-cache-nodes 1
```

5. **Create ECS Cluster & Services**
```bash
aws ecs create-cluster --cluster-name product-mgmt-prod

# Create task definitions and services
# (See AWS console or use CloudFormation templates)
```

## Heroku Deployment

### Quick Setup

```bash
# Login
heroku login

# Create app
heroku create product-mgmt-saas

# Add PostgreSQL
heroku addons:create heroku-postgresql:standard-0

# Add Redis
heroku addons:create heroku-redis:premium-0

# Set environment variables
heroku config:set JWT_SECRET=$(openssl rand -base64 32)
heroku config:set JWT_REFRESH_SECRET=$(openssl rand -base64 32)
heroku config:set NODE_ENV=production

# Deploy
git push heroku main

# Run migrations
heroku run npm run migrate

# View logs
heroku logs --tail
```

### Procfile Configuration

```procfile
web: npm run start
worker: npm run worker
```

## Docker Production Setup

### Environment Configuration

```bash
# .env.production
NODE_ENV=production
PORT=5000
DB_HOST=postgres.prod.example.com
DB_NAME=product_management
DB_USER=prod_user
DB_PASSWORD=SecurePassword123!
JWT_SECRET=<32-char-random-string>
JWT_REFRESH_SECRET=<32-char-random-string>
CORS_ORIGIN=https://yourdomain.com
REDIS_HOST=redis.prod.example.com
```

### Docker Compose Production

```yaml
version: '3.9'

services:
  postgres:
    image: postgres:15-alpine
    restart: always
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./backups:/backups
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    restart: always
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    networks:
      - app-network

  backend:
    image: product-mgmt-backend:latest
    restart: always
    environment:
      NODE_ENV: production
      DATABASE_URL: postgresql://${DB_USER}:${DB_PASSWORD}@postgres:5432/${DB_NAME}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  frontend:
    image: product-mgmt-frontend:latest
    restart: always
    depends_on:
      - backend
    networks:
      - app-network

volumes:
  postgres_data:
  redis_data:

networks:
  app-network:
    driver: bridge
```

## Database Backup Strategy

### Automated Daily Backups

```bash
#!/bin/bash
# backup.sh
BACKUP_DIR="/backups/db_backups"
DATE=$(date +%Y%m%d_%H%M%S)
FILENAME="backup_${DATE}.sql"

mkdir -p $BACKUP_DIR
pg_dump -h $DB_HOST -U $DB_USER $DB_NAME | gzip > "$BACKUP_DIR/$FILENAME.gz"

# Keep last 30 days only
find $BACKUP_DIR -name "*.gz" -mtime +30 -delete

# Upload to S3
aws s3 cp "$BACKUP_DIR/$FILENAME.gz" s3://backup-bucket/db-backups/

echo "Backup completed: $FILENAME"
```

Add to crontab:
```cron
0 2 * * * /home/app/backup.sh >> /var/log/backup.log 2>&1
```

### Recovery

```bash
# Download backup
aws s3 cp s3://backup-bucket/db-backups/backup_20240514_020000.sql.gz .

# Restore
gunzip backup_20240514_020000.sql.gz
psql -h $DB_HOST -U $DB_USER $DB_NAME < backup_20240514_020000.sql
```

## Monitoring & Logging

### CloudWatch Setup

```javascript
const WinstonCloudWatch = require('winston-cloudwatch');

const logger = winston.createLogger({
  transports: [
    new WinstonCloudWatch({
      logGroupName: 'product-mgmt-prod',
      logStreamName: 'backend',
      awsRegion: 'us-east-1'
    })
  ]
});
```

### Application Metrics

- CPU Usage
- Memory Consumption
- Database Connections
- API Response Time
- Error Rate
- Active Users

## Security Hardening

### SSL/TLS Configuration

```javascript
if (process.env.NODE_ENV === 'production') {
  app.use((req, res, next) => {
    if (req.header('x-forwarded-proto') !== 'https') {
      res.redirect(`https://${req.header('host')}${req.url}`);
    } else {
      next();
    }
  });
}
```

### Rate Limiting

```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100 // 100 requests per windowMs
});

app.use('/api/', limiter);
```

## CI/CD Pipeline

### GitHub Actions Workflow

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm install
      - run: npm test

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build and push Docker images
        run: |
          docker build -t product-mgmt-backend:latest backend/
          docker tag product-mgmt-backend:latest ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.us-east-1.amazonaws.com/product-mgmt-backend:latest
          docker push ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.us-east-1.amazonaws.com/product-mgmt-backend:latest

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Update ECS service
        run: |
          aws ecs update-service \
            --cluster product-mgmt-prod \
            --service backend-service \
            --force-new-deployment
```

## Performance Optimization

### Database Tuning

```sql
-- Analyze slow queries
EXPLAIN ANALYZE SELECT * FROM tasks WHERE project_id = 'xxx';

-- Update statistics
ANALYZE;
VACUUM ANALYZE;

-- Add indexes where needed
CREATE INDEX idx_created_desc ON tasks(created_at DESC);
```

### Caching Strategy

```javascript
const redis = require('redis');
const client = redis.createClient({
  host: process.env.REDIS_HOST,
  port: process.env.REDIS_PORT
});

// Cache products for 1 hour
const getCachedProducts = async () => {
  const cached = await client.get('products');
  if (cached) return JSON.parse(cached);
  
  const products = await db.query('SELECT * FROM products');
  await client.setex('products', 3600, JSON.stringify(products));
  return products;
};
```

---

**Production deployment complete! 🎉**

For support: support@example.com
