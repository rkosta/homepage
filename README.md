# Homepage Kubernetes Deployment

This repository contains Kubernetes manifest files for deploying the [Homepage](https://github.com/gethomepage/homepage) application - a modern, fully customizable application dashboard and service portal.

## Overview

Homepage is a self-hosted application dashboard that provides:
- Centralized access to internal services and bookmarks
- Real-time Kubernetes cluster and node monitoring
- Customizable widgets and service groups
- Integration with various services and APIs

This deployment is configured for the `teamnub.lan` domain and includes all necessary Kubernetes resources with RBAC permissions for cluster monitoring.

## Repository Structure

The manifests are numbered sequentially to indicate the recommended deployment order:

```
.
├── 01-homepage-service-account.yaml      # ServiceAccount for Homepage
├── 02-homepage-secret.yaml                # Service account token secret
├── 03-homepage-config-map.yaml            # Application configuration
├── 04-homepage-cluster-role.yaml          # RBAC permissions
├── 05-homepage-cluster-role-binding.yaml  # Role binding
├── 06-homepage-service.yaml               # Kubernetes Service
├── 07-homepage-deployment.yaml            # Application deployment
└── 08-homepage-ingress.yaml               # Traefik ingress configuration
```

## Features

### Kubernetes Integration
- **Read-only cluster access** via RBAC permissions
- Monitors pods, nodes, namespaces, and ingresses
- Supports standard Ingress, Traefik IngressRoutes, and Gateway API
- Access to metrics for resource monitoring

### Configuration
The ConfigMap includes:
- **Bookmarks**: Organized browser-style bookmarks
- **Services**: Grouped service links with descriptions
- **Widgets**: Dashboard widgets for cluster stats, resource monitoring, and search
- **Custom styling**: CSS and JavaScript customization files
- **Kubernetes mode**: Cluster information display

### Resource Allocation
- **Requests**: 100m CPU, 128Mi memory
- **Limits**: 500m CPU, 512Mi memory
- Single replica deployment

### Networking
- **Internal Service**: ClusterIP on port 3000
- **Ingress**: Accessible via Traefik at:
  - `homepage.teamnub.lan`
  - `homepage` (short hostname)

## Prerequisites

- Kubernetes cluster (tested on pikube/Team Nub cluster)
- Traefik ingress controller installed
- Metrics server (for resource monitoring widgets)

## Deployment

### Configure PiHole API Key

Before deploying, you need to configure the PiHole API key in the secrets file:

1. **Obtain your PiHole API key**:
   - Log into your PiHole admin interface
   - Navigate to Settings → API
   - Copy your API key

2. **Update the secret manifest**:

   Edit [00-homepage-secret.yaml](00-homepage-secret.yaml) and replace `YOUR_API_KEY_HERE` with your actual PiHole API key:

   ```bash
   # Option 1: Edit the file directly
   nano 00-homepage-secret.yaml
   ```

   Or use kubectl to create the secret directly without storing the key in the file:

   ```bash
   # Option 2: Create the secret from command line
   kubectl create secret generic homepage-secrets \
     --from-literal=PIHOLE_API_KEY='your-actual-api-key-here' \
     --dry-run=client -o yaml | kubectl apply -f -
   ```

3. **Apply the secret**:

   If you edited the file:
   ```bash
   kubectl apply -f 00-homepage-secret.yaml
   ```

   **Security Note**: Never commit the actual API key to version control. The placeholder value should remain in the repository.

### Quick Deploy

Deploy all manifests in order:

```bash
kubectl apply -f 01-homepage-service-account.yaml
kubectl apply -f 02-homepage-secret.yaml
kubectl apply -f 03-homepage-config-map.yaml
kubectl apply -f 04-homepage-cluster-role.yaml
kubectl apply -f 05-homepage-cluster-role-binding.yaml
kubectl apply -f 06-homepage-service.yaml
kubectl apply -f 07-homepage-deployment.yaml
kubectl apply -f 08-homepage-ingress.yaml
```

Or deploy all at once:

```bash
kubectl apply -f .
```

### Verify Deployment

```bash
# Check pod status
kubectl get pods -l app=homepage

# Check service
kubectl get svc homepage

# Check ingress
kubectl get ingress homepage
```

## Configuration

### Modifying Settings

The application configuration is stored in the ConfigMap ([03-homepage-config-map.yaml](03-homepage-config-map.yaml)). Edit this file to customize:

- **services.yaml**: Add or modify service links and groups
- **bookmarks.yaml**: Configure bookmarks
- **widgets.yaml**: Enable/disable widgets and configure their settings
- **settings.yaml**: Configure integrations (Longhorn, etc.)
- **custom.css**: Add custom styling
- **custom.js**: Add custom JavaScript

After modifying the ConfigMap, apply changes and restart the pod:

```bash
kubectl apply -f 03-homepage-config-map.yaml
kubectl rollout restart deployment/homepage
```

### Adding Services

Edit the `services.yaml` section in the ConfigMap to add new service groups:

```yaml
- Group Name:
    - Service Name:
        href: https://service.url
        description: Service description
        icon: icon-name.png
```

### Customizing Widgets

Edit the `widgets.yaml` section to add or modify dashboard widgets. Available widgets include:
- Kubernetes cluster and node information
- Resource monitoring (CPU, memory, network)
- Search engines
- And many more (see [Homepage documentation](https://gethomepage.dev/))

## RBAC Permissions

The deployment includes a ClusterRole with read-only permissions for:
- Namespaces, Pods, Nodes
- Ingresses (standard and Traefik IngressRoutes)
- Gateway API resources (HTTPRoutes, Gateways)
- Pod and Node metrics

These permissions allow Homepage to discover and monitor cluster resources without modification capabilities.

## Access

Once deployed, access Homepage at:
- `http://homepage.teamnub.lan` (via Traefik ingress)
- `http://homepage` (short hostname)

## Container Image

This deployment uses the official Homepage image:
- **Image**: `ghcr.io/gethomepage/homepage:latest`
- **Pull Policy**: Always (ensures latest version)
- **Port**: 3000 (HTTP)

## Troubleshooting

### Check pod logs
```bash
kubectl logs -l app=homepage
```

### Verify ConfigMap
```bash
kubectl describe configmap homepage
```

### Check RBAC permissions
```bash
kubectl auth can-i --list --as=system:serviceaccount:default:homepage
```

### Test service connectivity
```bash
kubectl port-forward svc/homepage 3000:3000
# Access at http://localhost:3000
```

## Upstream Project

This deployment is for the Homepage project:
- **GitHub**: https://github.com/gethomepage/homepage
- **Documentation**: https://gethomepage.dev/

## License

The Kubernetes manifests in this repository are provided as-is. The Homepage application itself is licensed under GPL-3.0 (see upstream project).

## Contributing

To modify or improve these manifests:
1. Edit the relevant YAML files
2. Test in a development cluster
3. Update this README if configuration changes
4. Submit changes via your preferred workflow

## Notes

- This configuration is tailored for the Team Nub (`teamnub.lan`) Kubernetes cluster
- The deployment uses `imagePullPolicy: Always` to ensure the latest image is pulled
- Logs are stored in an emptyDir volume at `/app/config/logs`
- The allowed hosts configuration includes `homepage`, `homepage.teamnub.lan`, and `pikube` hosts
