# Example: Kubernetes

This example shows how to run solidtime on Kubernetes with an external PostgreSQL database.

## Prerequisites

- [Kubernetes](https://kubernetes.io/) cluster (v1.19+)
- [kubectl](https://kubernetes.io/docs/tasks/tools/) configured to access your cluster
- [Nginx Ingress Controller](https://kubernetes.github.io/ingress-nginx/) installed in your cluster
- [Cert-Manager](https://cert-manager.io/) with a ClusterIssuer configured (e.g., `letsencrypt-cluster-issuer`)
- External PostgreSQL database
- Storage class that supports `ReadWriteMany` access mode (e.g., NFS, EFS, Longhorn)

## Installation

1. Copy the example files and adjust the configuration to your needs

```bash
# Configure database credentials and application key
nano 02-secrets.yaml

# Configure domain and application settings
nano 03-configmap.yaml

# Configure your domain and ClusterIssuer
nano 11-ingress.yaml

# Optionally: Configure storage class
nano 04-pvc.yaml
```

2. Generate an application key

```bash
docker run --rm solidtime/solidtime:latest php artisan key:generate --show
```

Copy the output and update `APP_KEY` in `02-secrets.yaml`.

3. Deploy to Kubernetes

```bash
kubectl apply -f .
```

4. Verify deployment

```bash
# Check all resources
kubectl get all -n solidtime

# Check certificate status
kubectl get certificate -n solidtime

# View logs
kubectl logs -n solidtime deployment/solidtime-app -f
```

Follow further instructions in [GUIDE.md](GUIDE.md) for detailed configuration, monitoring, and troubleshooting.

## Architecture

This deployment includes:

- **App Deployment**: Main HTTP application (2 replicas) with automatic database migrations
- **Queue Worker**: Background job processor for async tasks
- **Scheduler**: CronJob running Laravel scheduler every minute
- **Gotenberg**: PDF generation microservice
- **Ingress**: HTTPS with automatic TLS certificate provisioning

## Storage

The deployment mounts a shared PersistentVolumeClaim at `/var/www/html/storage/app` to persist user uploads while preserving Laravel's framework directories from the container image.

## Important Notes

- Database migrations run automatically via init container on deployment
- The application requires a PostgreSQL database (not included in manifests)
- All sensitive data uses `stringData` in secrets (automatically base64 encoded by Kubernetes)
- Default resource limits are set for production use

## Further Reading

See [GUIDE.md](GUIDE.md) for:
- Detailed architecture documentation
- Complete configuration reference
- Monitoring and maintenance commands
- Troubleshooting guide
- Security best practices

