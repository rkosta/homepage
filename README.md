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
├── 00-homepage-sealed-secret.yaml        # Sealed Secrets for service credentials
├── 00-homepage-secret.yaml               # Secret template (DO NOT apply directly)
├── 01-homepage-service-account.yaml      # ServiceAccount for Homepage
├── 02-homepage-secret.yaml               # Service account token secret
├── 03-homepage-config-map.yaml           # Application configuration
├── 04-homepage-cluster-role.yaml         # RBAC permissions
├── 05-homepage-cluster-role-binding.yaml # Role binding
├── 06-homepage-service.yaml              # Kubernetes Service
├── 07-homepage-deployment.yaml           # Application deployment
└── 08-homepage-ingress.yaml              # Traefik ingress configuration
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
- **Proxmox integration**: Hypervisor monitoring and VM/container statistics
- **Service integrations**: PiHole, OpenMediaVault, and Home Assistant widgets

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

### Configure Service Credentials

This deployment uses Sealed Secrets for secure credential management. The encrypted secrets are stored in [00-homepage-sealed-secret.yaml](00-homepage-sealed-secret.yaml).

#### Understanding Sealed Secrets

- **SealedSecret**: Encrypted secrets that can be safely stored in version control
- **Regular Secret**: The [00-homepage-secret.yaml](00-homepage-secret.yaml) file is a template showing the structure but should NOT be applied directly
- The SealedSecret controller will decrypt and create the actual Secret when applied to the cluster

#### Required Credentials

The following credentials are encrypted in the SealedSecret:

1. **PiHole API Key** (`PIHOLE_API_KEY`)
   - Obtain from your PiHole admin interface
   - Navigate to Settings → API

2. **OpenMediaVault Username** (`OMV_USERNAME`)
   - Typically `admin` or your OMV admin username

3. **OpenMediaVault Password** (`OMV_PASSWORD`)
   - Your OMV admin password

4. **Home Assistant Token** (`HA_TOKEN`)
   - Long-lived access token for Home Assistant API
   - Generate from your Home Assistant profile settings

5. **Proxmox Username** (`PROXMOX_USERNAME`)
   - API token ID in the format `user@realm!tokenid`
   - Example: `homepage@pve!homepage-api`

6. **Proxmox Password** (`PROXMOX_PASSWORD`)
   - The secret value for the Proxmox API token

#### How to Obtain Home Assistant Token

To generate a long-lived access token for Home Assistant:

1. Log in to your Home Assistant instance
2. Click on your profile (bottom left corner)
3. Scroll down to the "Long-Lived Access Tokens" section
4. Click "Create Token"
5. Give it a descriptive name (e.g., "Homepage Dashboard")
6. Copy the generated token immediately (it won't be shown again)

This token will be used in the ConfigMap to authenticate Homepage's requests to Home Assistant's API for displaying sensor data, entity states, and other information.

#### How to Obtain Proxmox API Token

To create an API token for Homepage in Proxmox VE:

1. Log in to your Proxmox VE web interface
2. Navigate to **Datacenter → Permissions → API Tokens**
3. Click **Add** to create a new API token
4. Configure the token:
   - **User**: Select or create a user (e.g., `homepage@pve`)
   - **Token ID**: Give it a descriptive name (e.g., `homepage-api`)
   - **Privilege Separation**: Uncheck to use user permissions
   - **Expire**: Set expiration date or leave empty for no expiration
5. Click **Add** and **copy the secret immediately** (it won't be shown again)
6. The username format will be: `username@realm!tokenid` (e.g., `homepage@pve!homepage-api`)

**Required Permissions:**
The Proxmox user needs the following permissions:
- `VM.Audit` - View VM/LXC configurations and status
- `Datastore.Audit` - View storage information
- `Sys.Audit` - View node information

You can create a custom role or use the built-in `PVEAuditor` role for read-only access.

**Setting Permissions:**
```bash
# Create a user (if not exists)
pveum user add homepage@pve

# Create API token
pveum user token add homepage@pve homepage-api

# Grant PVEAuditor role to the user at datacenter level
pveum acl modify / -user homepage@pve -role PVEAuditor
```

#### Updating Secrets

If you need to update the credentials, you'll need to:

1. Create a new sealed secret using `kubeseal`:
```bash
kubectl create secret generic homepage-secrets \
  --from-literal=PIHOLE_API_KEY='your-actual-api-key-here' \
  --from-literal=OMV_USERNAME='your-omv-username' \
  --from-literal=OMV_PASSWORD='your-omv-password' \
  --from-literal=HA_TOKEN='your-home-assistant-token-here' \
  --from-literal=PROXMOX_USERNAME='homepage@pve!homepage-api' \
  --from-literal=PROXMOX_PASSWORD='your-proxmox-api-token-secret' \
  --dry-run=client -o yaml | \
  kubeseal -o yaml > 00-homepage-sealed-secret.yaml
```

2. Apply the updated SealedSecret:
```bash
kubectl apply -f 00-homepage-sealed-secret.yaml
```

**Security Note**: The SealedSecret is encrypted specifically for your cluster and can be safely committed to version control. Never commit the unencrypted [00-homepage-secret.yaml](00-homepage-secret.yaml) with actual credentials.

### Quick Deploy

Deploy all manifests in order:

```bash
kubectl apply -f 00-homepage-sealed-secret.yaml
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

### Proxmox Integration

This deployment includes comprehensive Proxmox VE integration configured in the ConfigMap.

#### Global Proxmox Configuration

The `proxmox.yaml` section in the ConfigMap defines the global connection settings:

```yaml
proxmox.yaml: |
  pve:
    url: https://pve.teamnub.lan:8006
    token: {{HOMEPAGE_VAR_PROXMOX_USERNAME}}
    secret: {{HOMEPAGE_VAR_PROXMOX_PASSWORD}}
```

This configuration is automatically populated from the sealed secrets and provides centralized authentication.

#### Proxmox Widget

Add a Proxmox widget to display overall hypervisor statistics:

```yaml
- Hypervisors:
    - Proxmox VE:
        href: https://pve.teamnub.lan:8006
        description: Virtualization Platform
        icon: proxmox.svg
        widget:
          type: proxmox
          url: https://pve.teamnub.lan:8006
          username: {{HOMEPAGE_VAR_PROXMOX_USERNAME}}
          password: {{HOMEPAGE_VAR_PROXMOX_PASSWORD}}
```

This widget displays:
- Cluster resource usage (CPU, memory, storage)
- Number of VMs and containers
- Node status

#### Per-VM/Container Monitoring

You can also monitor individual VMs or LXC containers by adding Proxmox-specific fields to service entries:

```yaml
- VMs:
  - HomeAssistant:
      icon: home-assistant.png
      href: http://haos.teamnub.lan:8123/
      description: Home automation
      siteMonitor: http://haos.teamnub.lan:8123/
      proxmoxNode: pve          # Proxmox node name
      proxmoxVMID: 102          # VM or container ID
      showStats: true           # Display resource statistics
```

This configuration will show:
- VM/container status (running, stopped)
- CPU and memory usage
- Uptime
- Network traffic

To find your VMID, check the Proxmox web interface or run:
```bash
# On your Proxmox node
qm list        # For VMs
pct list       # For LXC containers
```

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
