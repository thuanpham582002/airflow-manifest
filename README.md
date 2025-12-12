# Apache Airflow Kubernetes Manifests

This repository contains Kubernetes manifests for deploying Apache Airflow with all its components using Kustomize.

## Architecture

The deployment includes a complete Airflow setup with:
- **Airflow API Server** - REST API and web interface
- **Airflow Scheduler** - DAG scheduling and execution
- **Airflow Worker** - Task execution
- **Airflow Triggerer** - Trigger-based DAG execution
- **Airflow DAG Processor** - DAG file processing
- **Redis** - Message broker and caching
- **PostgreSQL** - Metadata database (assumed existing)

## Components Overview

### Core Airflow Services
- **airflow-apiserver** - Web UI and REST API (ClusterIP on port 8080)
- **airflow-scheduler** - DAG scheduler component
- **airflow-worker** - Task execution workers
- **airflow-triggerer** - Trigger-based execution
- **airflow-dag-processor** - DAG file processing

### Infrastructure Services
- **redis** - Message broker for Celery
- **postgresql-setup** - Database initialization job

### Storage and Configuration
- **Persistent Volumes** - DAGs, logs, and plugins storage
- **ConfigMaps** - Airflow configuration
- **Secrets** - Encryption keys and database credentials
- **ServiceAccounts** - Kubernetes RBAC

## Prerequisites

1. **Kubernetes Cluster** with at least:
   - 4GB RAM minimum
   - 2+ CPU cores recommended

2. **PostgreSQL Database** (existing):
   - Database: `airflow`
   - User: `airflow`
   - Connection string configured in secrets

3. **Storage Class** for persistent volumes

## Deployment

### Quick Deploy
```bash
# Deploy Airflow with all components
kubectl apply -k .

# Check deployment status
kubectl get pods -n airflow
kubectl get services -n airflow

# Watch pods startup
kubectl get pods -n airflow -w
```

### Verify Setup
```bash
# Check database connection
kubectl logs -n airflow job/airflow-init-db -f

# Check scheduler logs
kubectl logs -n airflow deployment/airflow-scheduler -f

# Check API server logs
kubectl logs -n airflow deployment/airflow-apiserver -f
```

## Accessing Airflow

### Internal Access
- **Airflow Web UI**: `airflow-apiserver.airflow.svc.cluster.local:8080`
- **Airflow API**: `airflow-apiserver.airflow.svc.cluster.local:8080/api/v2/`
- **Redis**: `redis.airflow.svc.cluster.local:6379`

### External Access Options

Since only ClusterIP services are configured, use one of these methods:

1. **Port Forwarding** (recommended for temporary access):
```bash
# Forward Airflow web UI
kubectl port-forward -n airflow svc/airflow-apiserver 8080:8080
# Access at http://localhost:8080

# Forward Redis (if needed)
kubectl port-forward -n airflow svc/redis 6379:6379
```

2. **LoadBalancer Service** (if your cluster supports it):
```bash
kubectl patch svc airflow-apiserver -n airflow -p '{"spec":{"type":"LoadBalancer"}}'
```

3. **Ingress Controller** (manual setup):
Create your own Ingress resource to route traffic to the ClusterIP service

## Configuration

### Environment Variables
The Airflow deployment is configured with:

```yaml
AIRFLOW__CORE__EXECUTOR: CeleryExecutor
AIRFLOW__CORE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:password@postgresql:5432/airflow
AIRFLOW__CELERY__BROKER_URL: redis://redis:6379/0
AIRFLOW__CELERY__RESULT_BACKEND: redis://redis:6379/1
```

### Required Secrets

Before deployment, ensure these secrets are created:

1. **airflow-postgres-secret**:
   - `AIRFLOW_CONN_POSTGRES_DEFAULT`: PostgreSQL connection string

2. **airflow-fernet-key**:
   - `fernet-key`: Fernet encryption key for sensitive data

3. **airflow-webserver-secret**:
   - `webserver-secret-key`: Webserver session encryption

4. **airflow-api-secrets**:
   - `jwt-secret`: JWT authentication token
   - `internal-api-secret-key`: Internal API authentication

### Generate Secrets

```bash
# Generate Fernet key
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"

# Generate webserver secret key
python -c "import secrets; print(secrets.token_hex(32))"

# Generate JWT secret
python -c "import secrets; print(secrets.token_urlsafe(32))"

# Generate internal API secret
python -c "import secrets; print(secrets.token_urlsafe(32))"
```

## Storage Configuration

### Persistent Volumes
- **airflow-dags-pvc**: DAG files storage (default 1Gi)
- **airflow-logs-pvc**: Airflow logs storage (default 5Gi)
- **airflow-plugins-pvc**: Custom plugins storage (default 1Gi)

### DAG Management
Add DAG files to the persistent volume:
```bash
# Copy DAGs to running pod
kubectl cp your-dag.py airflow-apiserver-xxx:/opt/airflow/dags/

# Or mount a volume with your DAGs
```

## Scaling

### Horizontal Scaling
Scale workers to handle more tasks:
```bash
# Scale workers
kubectl scale deployment airflow-worker --replicas=5 -n airflow

# Scale triggerer for more concurrent triggers
kubectl scale deployment airflow-triggerer --replicas=2 -n airflow
```

### Resource Management
Update resource limits in the manifests based on your workload:
```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "2Gi"
    cpu: "1000m"
```

## Monitoring and Maintenance

### Health Checks
All components include health checks:
- **Liveness Probes**: Restart unhealthy containers
- **Readiness Probes**: Mark containers as ready for traffic

### Logs
```bash
# All Airflow component logs
kubectl logs -n airflow -l "app.kubernetes.io/part-of=airflow" -f

# Specific component logs
kubectl logs -n airflow deployment/airflow-scheduler -f
kubectl logs -n airflow deployment/airflow-worker -f
kubectl logs -n airflow deployment/airflow-apiserver -f
```

### Database Migration
```bash
# Run database migrations
kubectl exec -n airflow deployment/airflow-apiserver -- airflow db migrate
```

### Troubleshooting

#### Common Issues

1. **Database Connection Failed**
   ```bash
   # Check database connectivity
   kubectl exec -n airflow deployment/airflow-apiserver -- airflow db check
   ```

2. **Worker Not Connecting to Broker**
   ```bash
   # Check Redis connectivity
   kubectl exec -n airflow deployment/airflow-worker -- python -c "import redis; r=redis.Redis(host='redis', port=6379); print(r.ping())"
   ```

3. **DAGs Not Loading**
   ```bash
   # Check DAG directory
   kubectl exec -n airflow deployment/airflow-apiserver -- ls -la /opt/airflow/dags/

   # Check DAG processor logs
   kubectl logs -n airflow deployment/airflow-dag-processor -f
   ```

4. **Pod Issues**
   ```bash
   # Describe pod for issues
   kubectl describe pod -n airflow <pod-name>

   # Check pod events
   kubectl get events -n airflow --sort-by=.metadata.creationTimestamp
   ```

## Security Considerations

- **Network Policies**: Consider implementing network policies for Airflow services
- **Pod Security**: Security contexts are configured (runAsUser: 50000)
- **Secrets Management**: All sensitive data stored in Kubernetes secrets
- **RBAC**: Service accounts configured for minimal required permissions

## Customization

### Adding Custom Plugins
1. Create plugins in your repository
2. Mount to `/opt/airflow/plugins` volume
3. Update the `plugins` PVC size as needed

### Environment-Specific Configuration
Create overlays for different environments:
```yaml
# overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../../base
patchesStrategicMerge:
  - resources-patch.yaml
```

### Custom Executors
To use different executors, update the `airflow-configmap.yaml`:
```yaml
# For KubernetesExecutor
AIRFLOW__CORE__EXECUTOR: KubernetesExecutor

# For LocalExecutor
AIRFLOW__CORE__EXECUTOR: LocalExecutor
```

## Backup and Recovery

### Database Backup
```bash
# Backup PostgreSQL database
kubectl exec -n postgres deployment/postgresql -- pg_dump airflow > airflow-backup.sql
```

### DAGs and Plugins Backup
```bash
# Backup persistent volumes
kubectl cp airflow-apiserver-xxx:/opt/airflow/dags ./dags-backup/
kubectl cp airflow-apiserver-xxx:/opt/airflow/plugins ./plugins-backup/
```

## Cleanup

```bash
# Remove Airflow deployment
kubectl delete -k .

# Remove namespace (only if empty)
kubectl delete namespace airflow
```

## Notes

- **ClusterIP Only**: Services use ClusterIP for security
- **No External Exposure**: No Ingress or NodePort services configured
- **PostgreSQL Required**: Assumes existing PostgreSQL deployment
- **Persistent Storage**: Ensure sufficient storage for DAGs and logs
- **Secrets**: Update all secrets with production values before deployment

## Resources and Performance

### Minimum Resource Requirements
- **Total Memory**: 4GB+ (including all components)
- **Total CPU**: 2+ cores recommended
- **Storage**: 10GB+ for logs and DAGs

### Performance Tuning
- Adjust worker count based on task volume
- Scale scheduler/triggerer for high-throughput environments
- Monitor resource usage and adjust limits accordingly
- Consider larger persistent volumes for production workloads

This Airflow deployment is configured for production use with ClusterIP services for security and all necessary components for a complete Airflow installation.