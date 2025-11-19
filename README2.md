# Coworking Space Analytics Service - Deployment Guide

A Python Flask application providing business analytics for coworking space usage patterns, deployed on AWS EKS with automated CI/CD pipelines.


### Automated Build Process

The CI/CD pipeline is triggered on every push to the GitHub repository via CodeBuild webhooks:

```
Code Push → GitHub Webhook → CodeBuild → Docker Build → ECR Push → Manual Kubectl Apply
```

#### CodeBuild Configuration (buildspec.yaml)

The build process consists of three phases:

1. **Pre-Build Phase**
   - Authenticates to ECR using the build environment's IAM role
   - Generates three image tags for traceability:

2. **Build Phase**
   - Executes `docker build` from the `/analytics` directory
   - Uses `public.ecr.aws/docker/library/python:3.10-slim-bullseye` base image (avoids Docker Hub rate limiting)
   - Installs system dependencies (`build-essential`, `libpq-dev`) for PostgreSQL client compilation
   - Tags the resulting image with all three generated tags

3. **Post-Build Phase**
   - Pushes all image tags to the ECR repository
   - Outputs the full image URI for deployment reference

### Dockerfile Design

The Dockerfile uses a single-stage build optimized for production:


### Deployment Workflow

After a successful CodeBuild run, deployment to Kubernetes requires manual intervention for safety:

```bash
kubectl set image deployment/coworking \
  coworking=346395109265.dkr.ecr.us-east-1.amazonaws.com/coworking-analytics:<BUILD_NUMBER>


kubectl rollout status deployment/coworking

kubectl get pods -l app=coworking
kubectl logs -f deployment/coworking
```

## Project Structure

```
.
├── analytics/                      # Flask application code
│   ├── app.py                     # Main application entry point
│   ├── config.py                  # Database configuration
│   ├── requirements.txt           # Python dependencies
│   └── Dockerfile                 # Container image definition
│
├── db/                            # Database initialization scripts
│   ├── 1_create_tables.sql       # Schema definition (users, tokens tables)
│   ├── 2_seed_users.sql          # Test data: 1000 users
│   └── 3_seed_tokens.sql         # Test data: 1000 tokens
│
├── deployment/                    # Kubernetes manifests
│   ├── configmap.yaml            # ConfigMap (DB connection) + Secret (DB password)
│   ├── coworking.yaml            # Application Deployment + LoadBalancer Service
│   ├── postgresql-deployment.yaml # PostgreSQL Deployment
│   ├── postgresql-service.yaml    # PostgreSQL ClusterIP Service
│   ├── pvc.yaml                  # PersistentVolumeClaim for database storage
│   └── pv.yaml                   # PersistentVolume (for local development)
│
├── buildspec.yaml                # AWS CodeBuild pipeline configuration
├── README.md                     # Project overview
```

## Troubleshooting

### Pod Failures

```bash
kubectl get pods
kubectl describe pod <pod-name>

# View container logs
kubectl logs <pod-name>

```

### Database Connection Issues

```bash
# Verify PostgreSQL is running
kubectl get pods -l app=postgresql
kubectl logs -l app=postgresql

# Test database connectivity
kubectl exec -it <postgres-pod> -- psql -U myuser -d mydatabase -c '\dt'
kubectl exec -it <postgres-pod> -- psql -U myuser -d mydatabase -c 'SELECT COUNT(*) FROM users;'

# Check service endpoints
kubectl get endpoints postgresql-service
kubectl describe svc postgresql-service
```


### LoadBalancer Not Accessible

```bash
# Check LoadBalancer provisioning
kubectl get svc coworking
kubectl describe svc coworking

