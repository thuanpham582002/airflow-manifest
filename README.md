# Apache Airflow Kubernetes Manifests with Kustomize

This directory contains Kubernetes manifests for deploying Apache Airflow using Kustomize with CeleryExecutor for distributed execution.

## Directory Structure

```
airflow-manifests/
├── base/
│   ├── kustomization.yaml    # Base configuration
│   └── airflow.yaml          # All Airflow resources (Deployments, Services, PVCs, Jobs, RBAC)
├── overlays/
│   ├── dev/                  # Development environment overrides
│   │   ├── kustomization.yaml
│   │   ├── storage-patch.yaml        # Smaller storage sizes
│   │   ├── resources-patch.yaml      # Smaller resource limits
│   │   └── config-patch.yaml         # Dev-friendly configuration
│   └── prod/                 # Production environment overrides
│       ├── kustomization.yaml
│       ├── storage-patch.yaml        # Larger storage sizes
│       ├── resources-patch.yaml      # Larger resource limits
│       ├── replica-patch.yaml        # Stronger security secrets
│       └── config-patch.yaml         # Production-optimized configuration
└── README.md
```

## Components

The deployment includes the following components:

### Airflow Core Services
- **Web Server**: Web UI and REST API server
- **Scheduler**: Triggers DAGs and tasks based on schedule
- **Workers**: Execute tasks using CeleryExecutor
- **Triggerer**: Deferrable operators and triggers
- **Database Initialization Job**: Sets up database and creates admin user
- **Database Migration Job**: Runs database migrations

### Dependencies
- **PostgreSQL**: Primary metadata database
- **Redis**: Message broker for Celery workers

### Kubernetes Resources
- **ServiceAccount**: For Airflow to interact with Kubernetes API
- **RBAC**: Permissions for KubernetesExecutor (if needed)
- **PersistentVolumeClaims**: Storage for DAGs, logs, plugins, and database

## Deployment Options

### Development Environment

```bash
# Deploy to dev namespace
kubectl apply -k overlays/dev

# Check the deployment
kubectl get pods -n airflow-dev
kubectl get svc -n airflow-dev
```

### Production Environment

```bash
# Deploy to prod namespace
kubectl apply -k overlays/prod

# Check the deployment
kubectl get pods -n airflow-prod
kubectl get svc -n airflow-prod
```

### Base Configuration

```bash
# Deploy with base configuration
kubectl apply -k base

# Access Airflow Web UI using port-forward:
kubectl port-forward -n airflow svc/airflow-webserver-service 8080:8080
# Then visit: http://localhost:8080
# Default credentials: admin/admin
```

## Accessing Airflow

### Web UI (Port 8080)
- Internal cluster: `airflow-webserver-service.airflow.svc.cluster.local:8080`

### External Access Options
For external access, consider using:
- **Ingress**: Configure an Ingress controller with SSL termination
- **kubectl port-forward**: `kubectl port-forward -n namespace svc/airflow-webserver-service 8080:8080`
- **LoadBalancer**: Change service type to LoadBalancer if your cluster supports it

### Default Credentials
- Username: `admin`
- Password: `admin`

**Important**: Change the default credentials and secrets before using in production!
- Update the `airflow-secrets` and `airflow-webserver-secret` Secrets
- Generate new Fernet key and web server secret key

## Configuration

### Core Environment Variables

The manifests include these key configuration options:

#### Airflow Core Configuration
- `AIRFLOW__CORE__EXECUTOR`: CeleryExecutor for distributed execution
- `AIRFLOW__CORE__SQL_ALCHEMY_CONN`: PostgreSQL connection string
- `AIRFLOW__CORE__FERNET_KEY`: Encryption key for sensitive data
- `AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION`: Pause DAGs on creation
- `AIRFLOW__CORE__LOAD_EXAMPLES`: Load example DAGs (dev only)

#### Celery Configuration
- `AIRFLOW__CELERY__BROKER_URL`: Redis broker URL
- `AIRFLOW__CELERY__RESULT_BACKEND`: Redis result backend

#### Web Server Configuration
- `AIRFLOW__WEBSERVER__EXPOSE_CONFIG`: Expose configuration in UI (dev only)
- `AIRFLOW__WEBSERVER__AUTHENTICATE`: Enable authentication
- `AIRFLOW__WEBSERVER__SECRET_KEY`: Flask secret key for sessions

### Volume Mounts
- **DAGs**: `/opt/airflow/dags` - Store your DAG files here
- **Logs**: `/opt/airflow/logs` - Airflow execution logs
- **Plugins**: `/opt/airflow/plugins` - Custom plugins

### Customization

#### Environment-Specific Variables

Edit the patch files in overlays/dev or overlays/prod to customize:
- Storage sizes for database, DAGs, logs, and plugins
- Resource limits/requests for all components
- Replica counts for high availability
- Security configurations and secrets
- Logging levels and debugging options

#### Adding DAGs

1. **Copy DAG files to the PVC**:
```bash
kubectl cp your_dag.py airflow-dev/dag-pod-xxx:/opt/airflow/dags/
```

2. **Use Git sync** (recommended for production):
   - Add a sidecar container to sync DAGs from Git
   - Or use external DAG storage (S3, GCS, etc.)

3. **Build custom image**:
   - Create a custom Dockerfile with your DAGs
   - Update the image in the manifests

#### Adding Custom Providers

Build a custom Airflow image with your required packages:

```dockerfile
FROM apache/airflow:2.10.2-python3.11

# Install Python packages
RUN pip install apache-airflow-providers-cncf-kubernetes \
                apache-airflow-providers-amazon \
                apache-airflow-providers-google

# Copy your DAGs
COPY dags/ /opt/airflow/dags/
```

## Security Considerations

### Production Hardening

1. **Generate Strong Secrets**:
```bash
# Generate Fernet key
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"

# Generate Flask secret key
python -c "import secrets; print(secrets.token_hex(32))"
```

2. **Update Secrets**:
```bash
kubectl create secret generic airflow-secrets \
  --from-literal=AIRFLOW__CORE__FERNET_KEY=your-fernet-key \
  --from-literal=AIRFLOW__CORE__SQL_ALCHEMY_CONN=your-db-connection \
  --from-literal=POSTGRES_PASSWORD=your-db-password \
  --from-literal=POSTGRES_USER=your-db-user

kubectl create secret generic airflow-webserver-secret \
  --from-literal=AIRFLOW__WEBSERVER__SECRET_KEY=your-flask-secret
```

3. **Network Policies**: Implement network policies to restrict traffic
4. **RBAC**: Use proper RBAC for Kubernetes access and service accounts
5. **TLS**: Enable TLS for external access using Ingress with SSL
6. **Image Security**: Use private registry for production images

## Monitoring and Health Checks

### Health Checks
All deployments include health checks:
- **Liveness Probe**: Ensures the container is running
- **Readiness Probe**: Ensures the container is ready to serve traffic

### Logs
```bash
# Airflow component logs
kubectl logs -n namespace deployment/airflow-webserver -f
kubectl logs -n namespace deployment/airflow-scheduler -f
kubectl logs -n namespace deployment/airflow-worker -f
kubectl logs -n namespace deployment/airflow-triggerer -f

# Database logs
kubectl logs -n namespace deployment/postgresql -f

# Redis logs
kubectl logs -n namespace deployment/redis -f
```

## Scaling

### Horizontal Scaling
- **Web Servers**: For high availability (production: 2 replicas)
- **Schedulers**: For improved scheduling performance (production: 2 replicas)
- **Workers**: Scale based on task execution load (production: 5 replicas, dev: 1 replica)
- **Triggerers**: For deferrable operators (production: 2 replicas)

### Resource Scaling
- CPU and memory limits are configured per environment
- Adjust based on your workload and usage patterns
- Monitor resource usage and adjust accordingly

## Maintenance

### Database Maintenance
```bash
# Backup PostgreSQL
kubectl exec -n namespace deployment/postgresql -- \
  pg_dump -U airflow airflow > airflow_backup.sql

# Restore PostgreSQL
kubectl exec -i -n namespace deployment/postgresql -- \
  psql -U airflow airflow < airflow_backup.sql
```

### Airflow Maintenance
```bash
# Clean old logs
airflow db clean --clean-before-timestamp 2023-01-01

# Reset failed task instances
airflow tasks clear -d dag_id -s start_date -e end_date --state failed
```

### Upgrades
```bash
# Apply updates
kubectl apply -k overlays/environment

# Check rollout status
kubectl rollout status deployment/airflow-webserver -n namespace
```

## Troubleshooting

### Common Issues

1. **Database Connection Failed**: Check PostgreSQL connectivity and credentials
2. **Workers Not Connecting**: Verify Redis connection and Celery configuration
3. **DAGs Not Loading**: Check DAG folder permissions and file syntax
4. **High Memory Usage**: Adjust worker resources and monitor task complexity

### Debug Mode
In development environment, debug mode is enabled with:
- Exposed configuration in UI
- Disabled authentication
- DEBUG logging level

## Cleanup

```bash
# Delete dev deployment
kubectl delete -k overlays/dev

# Delete prod deployment
kubectl delete -k overlays/prod

# Delete base deployment
kubectl delete -k base

# Delete namespace (if empty)
kubectl delete namespace airflow-dev
kubectl delete namespace airflow-prod
kubectl delete namespace airflow
```

## Version Information

- **Apache Airflow**: `apache/airflow:2.10.2-python3.11`
- **PostgreSQL**: `postgres:15-alpine`
- **Redis**: `redis:7-alpine`

These manifests use pinned versions for better stability and security in production deployments.

## Additional Resources

- [Official Airflow Documentation](https://airflow.apache.org/docs/)
- [Airflow Kubernetes Executor Guide](https://airflow.apache.org/docs/apache-airflow-providers-cncf-kubernetes/stable/kubernetes_executor.html)
- [Airflow Helm Chart](https://github.com/apache/airflow/tree/main/chart)
- [Best Practices Guide](https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html)