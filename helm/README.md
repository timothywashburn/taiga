# Taiga Helm Chart

Kubernetes Helm chart for deploying Taiga project management platform.

## Prerequisites

- Kubernetes cluster (tested with v1.25+)
- Helm 3.x
- Traefik ingress controller
- cert-manager for TLS certificates
- A ClusterIssuer named `letsencrypt-prod` configured in cert-manager

## Architecture

This chart deploys the following components:

- **PostgreSQL**: Database (StatefulSet)
- **RabbitMQ**: Message queue for async tasks and events (StatefulSet)
- **Taiga Backend**: Django REST API server (Deployment)
- **Taiga Frontend**: Angular SPA application (Deployment)
- **Taiga Events**: WebSocket server for real-time updates (Deployment)
- **Taiga Protected**: Protected file download service (Deployment)
- **Taiga Gateway**: Nginx reverse proxy tying everything together (Deployment)

## Quick Start

1. Edit `values.yaml` and configure at minimum:
   - `ingress.host`: Your domain name (e.g., `taiga.example.com`)
   - `taiga.secretKey`: A random secret key (generate with `openssl rand -hex 32`)

2. Install the chart:
   ```bash
   helm install taiga . -n taiga --create-namespace
   ```

3. Wait for all pods to be ready:
   ```bash
   kubectl get pods -n taiga
   ```

4. Create a superuser:
   ```bash
   kubectl exec -it deployment/taiga-back -n taiga -- python manage.py createsuperuser
   ```

5. Access Taiga at `https://your-domain.com`

## Configuration

### Required Configuration

Edit `values.yaml` and set these values:

```yaml
ingress:
  host: taiga.example.com  # Your domain name

taiga:
  secretKey: "your-random-secret-key-here"  # Generate with: openssl rand -hex 32
```

### Security Configuration

**IMPORTANT**: For production deployments, you should change the default passwords:

```yaml
postgres:
  user: taiga
  password: CHANGE_THIS_PASSWORD  # Use a strong password

rabbitmq:
  user: taiga
  password: CHANGE_THIS_PASSWORD  # Use a strong password
  erlangCookie: CHANGE_THIS_COOKIE  # Generate with: openssl rand -hex 32
```

### Email Configuration

To enable email notifications (recommended for production):

```yaml
taiga:
  email:
    backend: smtp  # Change from 'console' to 'smtp'
    host: smtp.example.com
    port: 587
    user: your-smtp-user
    password: your-smtp-password
    defaultFrom: noreply@example.com
    useTLS: true
    useSSL: false
```

### Public Registration

By default, public registration is disabled. To enable it:

```yaml
taiga:
  publicRegisterEnabled: true
```

### Storage Configuration

Configure persistent storage sizes and optionally bind to specific PVs:

```yaml
postgres:
  storage:
    volumeName: ""  # Optional: specify PV name
    size: 5Gi

rabbitmq:
  storage:
    volumeName: ""
    size: 1Gi

media:
  storage:
    volumeName: ""
    size: 5Gi  # Stores user uploads and attachments
```

### Image Versions

By default, the chart uses `latest` tags. For production, specify specific versions:

```yaml
image:
  backend:
    name: taigaio/taiga-back
    tag: "6.6.0"  # Pin to specific version
  frontend:
    name: taigaio/taiga-front
    tag: "6.6.0"
  events:
    name: taigaio/taiga-events
    tag: "6.6.0"
  protected:
    name: taigaio/taiga-protected
    tag: "6.6.0"
```

## Installation

### Install to a specific namespace

```bash
helm install taiga . -n taiga --create-namespace
```

### Upgrade an existing installation

```bash
helm upgrade taiga . -n taiga
```

### Uninstall

```bash
helm uninstall taiga -n taiga
```

## Post-Installation

### Create a superuser

```bash
kubectl exec -it deployment/taiga-back -n taiga -- python manage.py createsuperuser
```

Follow the prompts to create your admin account.

### Access the admin panel

Navigate to `https://your-domain.com/admin/` and login with your superuser credentials.

## Management Commands

You can run Django management commands using:

```bash
kubectl exec -it deployment/taiga-back -n taiga -- python manage.py <command>
```

Useful commands:
- `createsuperuser` - Create a superuser account
- `changepassword <username>` - Change a user's password
- `shell` - Open Django shell
- `dbshell` - Open database shell

## Troubleshooting

### Check pod status

```bash
kubectl get pods -n taiga
```

### View logs

```bash
# Backend logs
kubectl logs -f deployment/taiga-back -n taiga

# Frontend logs
kubectl logs -f deployment/taiga-front -n taiga

# Gateway logs
kubectl logs -f deployment/taiga-gateway -n taiga

# Events logs
kubectl logs -f deployment/taiga-events -n taiga

# Database logs
kubectl logs -f statefulset/postgres -n taiga
```

### Common Issues

**Issue**: Pods stuck in `Pending` state
- Check PVC status: `kubectl get pvc -n taiga`
- Ensure your cluster has a default StorageClass or specify `volumeName` in values.yaml

**Issue**: Database connection errors
- Ensure PostgreSQL pod is running and ready
- Check database credentials in values.yaml
- Verify RabbitMQ is also running

**Issue**: TLS certificate not issued
- Verify cert-manager is installed: `kubectl get pods -n cert-manager`
- Check certificate status: `kubectl get certificate -n taiga`
- Verify ClusterIssuer exists: `kubectl get clusterissuer letsencrypt-prod`

## Advanced Configuration

### OAuth/SSO Integration

To enable GitHub/GitLab authentication, edit `templates/secrets.yaml` and add the OAuth credentials, then update the backend deployment environment variables. Refer to the Taiga documentation for detailed OAuth setup.

### Custom Domain with Subpath

If you want to serve Taiga under a subpath (e.g., `example.com/taiga`), you'll need to:
1. Update `TAIGA_SUBPATH` environment variable in backend deployment
2. Update `SUBPATH` in frontend deployment
3. Modify nginx configuration in gateway configmap
4. Update ingress rules

## Backup and Restore

### Backup PostgreSQL

```bash
kubectl exec -it postgres-0 -n taiga -- pg_dump -U taiga taiga > taiga-backup.sql
```

### Restore PostgreSQL

```bash
kubectl exec -i postgres-0 -n taiga -- psql -U taiga taiga < taiga-backup.sql
```

### Backup Media Files

```bash
kubectl cp taiga/taiga-back-<pod-id>:/taiga-back/media ./media-backup
```

## Monitoring

Consider setting up monitoring for:
- Pod health and resource usage
- PostgreSQL performance
- RabbitMQ queue lengths
- Nginx access logs

## Security Considerations

1. Change all default passwords in production
2. Use specific image tags instead of `latest`
3. Configure network policies to restrict pod communication
4. Enable and configure SMTP for proper email notifications
5. Regularly update Taiga images for security patches
6. Consider using external managed databases (PostgreSQL/RabbitMQ) for production

## Support

For Taiga-specific issues, refer to:
- Taiga Documentation: https://docs.taiga.io/
- Taiga Community: https://community.taiga.io/
- GitHub Issues: https://github.com/taigaio/taiga

For Kubernetes/Helm issues with this chart, check the pod logs and Kubernetes events.
