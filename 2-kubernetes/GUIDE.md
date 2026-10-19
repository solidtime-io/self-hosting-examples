# Solidtime Kubernetes Deployment

This repository contains Kubernetes manifests for deploying [Solidtime](https://github.com/solidtime-io/solidtime) - a modern time tracking application - to a Kubernetes cluster.

## 📋 Prerequisites

Before deploying Solidtime to Kubernetes, ensure you have:

- A Kubernetes cluster (v1.19+)
- `kubectl` configured to access your cluster
- **Nginx Ingress Controller** installed in your cluster
- **Cert-Manager** with a ClusterIssuer configured (e.g., `letsencrypt-cluster-issuer`)
- An **external PostgreSQL database** (the manifests connect to an external DB)
- A **storage class** that supports `ReadWriteMany` access mode (for shared storage between pods)

## 🏗️ Architecture

This deployment consists of the following components:

### Application Components

1. **App Deployment** (`07-app-deployment.yaml`)
   - Main HTTP application serving the web interface
   - Runs on port 8000
   - Scaled to 2 replicas for high availability
   - Health checks at `/health-check/up`
   - Init container automatically runs database migrations on deployment

2. **Queue Worker Deployment** (`09-queue-deployment.yaml`)
   - Background job processor using Laravel queue
   - Processes async tasks (emails, reports, etc.)
   - Single replica (can be scaled if needed)

3. **Scheduler CronJob** (`10-scheduler-cronjob.yaml`)
   - Runs Laravel scheduler every minute
   - Executes `php artisan schedule:run`
   - Handles scheduled tasks (reminders, cleanup, etc.)

4. **Gotenberg Service** (`05-gotenberg-deployment.yaml`, `06-gotenberg-service.yaml`)
   - PDF generation microservice
   - Used for exporting reports and timesheets
   - Internal service only

### Infrastructure Components

- **Namespace** (`01-namespace.yaml`) - Isolated namespace `solidtime`
- **ConfigMap** (`03-configmap.yaml`) - Non-sensitive configuration
- **Secret** (`02-secrets.yaml`) - Sensitive data (DB credentials, APP_KEY, etc.)
- **PersistentVolumeClaim** (`04-pvc.yaml`) - Shared storage for uploads and logs
- **Service** (`08-app-service.yaml`) - ClusterIP service for the app
- **Ingress** (`11-ingress.yaml`) - HTTPS ingress with automatic TLS

## 🚀 Quick Start

### 1. Navigate to Directory

```bash
cd 2-kubernetes
```

### 2. Configure Your Environment

Edit the following files with your specific configuration:

#### `02-secrets.yaml`
```yaml
# Update these values:
DB_HOST: "your-postgres-host"          # Your PostgreSQL host
DB_DATABASE: "solidtime"                # Your database name
DB_USERNAME: "solidtime"                # Your database user
DB_PASSWORD: "your-secure-password"    # Your database password
APP_KEY: "base64:YOUR_APP_KEY_HERE"    # Generate with: docker run --rm solidtime/solidtime php artisan key:generate --show
```

**Note on Secrets**: The file uses `stringData` which accepts plain text values. Kubernetes automatically base64 encodes them when the secret is created. If you need to use `data` instead, manually base64 encode values: `echo -n "value" | base64`

#### `03-configmap.yaml`
```yaml
# Update these values:
APP_URL: "https://solidtime.example.com"  # Your domain
APP_TIMEZONE: "UTC"                        # Your timezone
APP_LOCALE: "en"                           # Your locale
```

#### `11-ingress.yaml`
```yaml
# Update these values:
host: solidtime.example.com  # Your domain (appears in 2 places)
cert-manager.io/cluster-issuer: "letsencrypt-cluster-issuer"  # Your ClusterIssuer name
```

#### `04-pvc.yaml`
```yaml
# Optionally specify your storage class:
storageClassName: your-storage-class  # Uncomment and set if needed
```

### 3. Generate Application Key

Generate a Laravel application key:

```bash
docker run --rm solidtime/solidtime:latest php artisan key:generate --show
```

Copy the output (e.g., `base64:xxxxx...`) and update `APP_KEY` in `02-secrets.yaml`.

### 4. Deploy to Kubernetes

Deploy all manifests in order:

```bash
kubectl apply -f 01-namespace.yaml
kubectl apply -f 02-secrets.yaml
kubectl apply -f 03-configmap.yaml
kubectl apply -f 04-pvc.yaml
kubectl apply -f 05-gotenberg-deployment.yaml
kubectl apply -f 06-gotenberg-service.yaml
kubectl apply -f 07-app-deployment.yaml
kubectl apply -f 08-app-service.yaml
kubectl apply -f 09-queue-deployment.yaml
kubectl apply -f 10-scheduler-cronjob.yaml
kubectl apply -f 11-ingress.yaml
```

Or apply all at once:

```bash
kubectl apply -f .
```

### 5. Verify Deployment

Check that all pods are running:

```bash
kubectl get pods -n solidtime
```

**Note**: The app deployment includes an init container that automatically runs `php artisan migrate --force` before the app starts, so migrations are applied automatically on each deployment.

### 6. Create First User (Optional)

You can create an admin user via the web interface or using artisan:

```bash
kubectl exec -n solidtime deployment/solidtime-app -- php artisan user:create
```

## 🔧 Configuration

### Environment Variables

All configuration is managed through:
- **ConfigMap** (`03-configmap.yaml`) - Non-sensitive settings
- **Secret** (`02-secrets.yaml`) - Sensitive credentials (using `stringData` for plain text values)

Key configuration areas:

#### Database Connection
```yaml
DB_CONNECTION: pgsql
DB_HOST: "your-postgres-host"
DB_PORT: "5432"
DB_DATABASE: "solidtime"
DB_USERNAME: "solidtime"
DB_PASSWORD: "your-password"
```

#### Mail Configuration (Optional)
```yaml
MAIL_MAILER: smtp
MAIL_HOST: "smtp.example.com"
MAIL_PORT: "587"
MAIL_USERNAME: "your-username"
MAIL_PASSWORD: "your-password"
MAIL_ENCRYPTION: "tls"
MAIL_FROM_ADDRESS: "noreply@example.com"
```

#### Application Settings
```yaml
APP_NAME: "Solidtime"
APP_ENV: "production"
APP_DEBUG: "false"
APP_URL: "https://solidtime.example.com"
```

#### Internal Service Communication
Services within the cluster communicate using Kubernetes DNS:
```
SERVICENAME.NAMESPACE.svc.cluster.local:PORT
```

For example, the Gotenberg service is accessed at:
```yaml
GOTENBERG_HOST: "gotenberg.solidtime.svc.cluster.local:3000"
```

**Note on Ports**: The port number is required in the connection string because Kubernetes DNS only resolves the hostname to the service's ClusterIP. The port information is not included in DNS resolution, so your application must specify it. You can only omit the port if:
- The service uses a standard port (80 for HTTP, 443 for HTTPS)
- Your application has a default port configured

**Note on Protocol**: Whether to include `http://` or `https://` depends on the application's expectations. Some apps handle this internally, while others require it in the connection string.

### Resource Limits

Default resource allocations:

| Component | Requests | Limits |
|-----------|----------|--------|
| App | 256Mi / 100m | 512Mi / 1000m |
| Queue | 256Mi / 100m | 512Mi / 500m |
| Scheduler | 128Mi / 50m | 256Mi / 200m |
| Gotenberg | 256Mi / 100m | 512Mi / 500m |

Adjust these in the respective deployment files based on your workload.

## 📊 Monitoring and Maintenance

### Check Deployment Status

```bash
# All resources
kubectl get all -n solidtime

# Pods
kubectl get pods -n solidtime

# Ingress
kubectl get ingress -n solidtime

# CronJob status
kubectl get cronjobs -n solidtime
```

### View Logs

```bash
# App logs
kubectl logs -n solidtime deployment/solidtime-app -f

# Queue worker logs
kubectl logs -n solidtime deployment/solidtime-queue -f

# Scheduler logs (last job)
kubectl logs -n solidtime job/solidtime-scheduler-<timestamp>

# Gotenberg logs
kubectl logs -n solidtime deployment/solidtime-gotenberg -f
```

### Scale Components

```bash
# Scale app replicas
kubectl scale -n solidtime deployment/solidtime-app --replicas=3

# Scale queue workers
kubectl scale -n solidtime deployment/solidtime-queue --replicas=2
```

### Update Application

```bash
# Update to a specific version
kubectl set image -n solidtime deployment/solidtime-app app=solidtime/solidtime:v1.2.3
kubectl set image -n solidtime deployment/solidtime-queue queue=solidtime/solidtime:v1.2.3

# Or update the image in the YAML files and reapply
kubectl apply -f 07-app-deployment.yaml
kubectl apply -f 09-queue-deployment.yaml
```

### Run Artisan Commands

```bash
# General syntax
kubectl exec -n solidtime deployment/solidtime-app -- php artisan <command>

# Examples:
kubectl exec -n solidtime deployment/solidtime-app -- php /var/www/html/artisan migrate
kubectl exec -n solidtime deployment/solidtime-app -- php /var/www/html/artisan cache:clear
kubectl exec -n solidtime deployment/solidtime-app -- php /var/www/html/artisan config:clear
kubectl exec -n solidtime deployment/solidtime-app -- php /var/www/html/artisan queue:restart
```

## 🔒 Security Considerations

1. **Secrets Management**: The `02-secrets.yaml` file contains sensitive data. Consider using:
   - Kubernetes Secrets (encrypted at rest)
   - External secret managers (e.g., HashiCorp Vault, AWS Secrets Manager)
   - Sealed Secrets for GitOps workflows

2. **Network Policies**: Consider implementing NetworkPolicies to restrict pod-to-pod communication

3. **RBAC**: Implement proper RBAC policies for accessing the namespace

4. **Image Security**: 
   - Use specific image tags instead of `latest` in production
   - Scan images for vulnerabilities
   - Use private registries if needed

5. **TLS Certificates**: The ingress uses cert-manager for automatic TLS. Ensure your ClusterIssuer is properly configured.

## 🐛 Troubleshooting

### Pods Not Starting

```bash
# Check pod status
kubectl describe pod -n solidtime <pod-name>

# Check events
kubectl get events -n solidtime --sort-by='.lastTimestamp'
```

### Database Connection Issues

```bash
# Test database connectivity from a pod
kubectl exec -n solidtime deployment/solidtime-app -- php /var/www/html/artisan migrate:status
```

### Storage Issues

```bash
# Check PVC status
kubectl get pvc -n solidtime

# Check if volumes are mounted
kubectl describe pod -n solidtime <pod-name>
```

### Ingress Not Working

```bash
# Check ingress status
kubectl describe ingress -n solidtime solidtime-ingress

# Check cert-manager certificate
kubectl get certificate -n solidtime

# Check nginx ingress logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```

### Scheduler Not Running

```bash
# Check CronJob
kubectl get cronjobs -n solidtime

# Check recent jobs
kubectl get jobs -n solidtime

# Manually trigger a job
kubectl create job -n solidtime --from=cronjob/solidtime-scheduler manual-run-1
```

## 📚 Additional Resources

- [Solidtime Official Documentation](https://docs.solidtime.io)
- [Solidtime GitHub Repository](https://github.com/solidtime-io/solidtime)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Nginx Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
- [Cert-Manager Documentation](https://cert-manager.io/docs/)

## 🤝 Contributing

Contributions are welcome! If you find issues or have improvements:

1. Fork the repository
2. Create your feature branch
3. Test your changes
4. Submit a pull request

## 📝 License

These Kubernetes manifests are provided as-is. Solidtime itself is licensed under the AGPL-3.0 license.

## ⚠️ Disclaimer

This is a community-maintained deployment configuration. Always test thoroughly in a staging environment before deploying to production.

## 💡 Tips

- **Backups**: Regularly backup your PostgreSQL database and PersistentVolumes
- **Monitoring**: Consider adding Prometheus metrics and Grafana dashboards
- **Logging**: Integrate with a centralized logging solution (ELK, Loki, etc.)
- **High Availability**: For production, ensure your PostgreSQL database has HA/failover configured
- **Resource Tuning**: Monitor resource usage and adjust requests/limits accordingly

---

**Need Help?** Open an issue or reach out to the Solidtime community!

