# koreader-sync-server-k8s

Helm chart for deploying [KOReader sync server](https://github.com/koreader/koreader-sync-server) (`kosync`) on Kubernetes. Bundles Redis inside the same pod.

## Prerequisites

- Kubernetes cluster
- Helm 3
- Istio (optional, for VirtualService routing)

## Install

```bash
helm install kosync ./charts/kosync
```

With custom values:

```bash
helm install kosync ./charts/kosync -f my-values.yaml
```

## Values

| Key | Default | Description |
|-----|---------|-------------|
| `replicaCount` | `1` | Number of replicas |
| `image.repository` | `koreader/kosync` | Container image |
| `image.tag` | `latest` | Image tag |
| `image.pullPolicy` | `IfNotPresent` | Pull policy |
| `service.type` | `ClusterIP` | Service type |
| `service.port` | `17200` | Service port |
| `virtualService.enabled` | `true` | Create Istio VirtualService |
| `virtualService.host` | `kosync.example.com` | Hostname for VirtualService |
| `virtualService.gateway` | `istio-system/gateway` | Istio Gateway reference |
| `persistence.enabled` | `true` | Enable PVCs |
| `persistence.storageClass` | `""` | StorageClass (empty = cluster default) |
| `persistence.accessMode` | `ReadWriteOnce` | PVC access mode |
| `persistence.redis.data.size` | `1Gi` | Redis data volume size |
| `persistence.redis.logs.size` | `500Mi` | Redis logs volume size |
| `persistence.app.logs.size` | `500Mi` | App logs volume size |
| `resources` | `{}` | CPU/memory resource requests and limits |
| `nodeSelector` | `{}` | Node selector |
| `tolerations` | `[]` | Tolerations |
| `affinity` | `{}` | Affinity rules |
| `podAnnotations` | `{}` | Pod annotations |
| `podSecurityContext` | `{}` | Pod security context |
| `securityContext` | `{}` | Container security context |

## Volumes

Three PVCs are created when `persistence.enabled: true`:

| Mount path | PVC suffix | Default size |
|-----------|------------|--------------|
| `/var/lib/redis` | `-redis-data` | 1Gi |
| `/var/log/redis` | `-redis-logs` | 500Mi |
| `/app/koreader-sync-server/logs` | `-app-logs` | 500Mi |

## Usage

### Create user

Password must be MD5-hashed before sending.

```bash
USERNAME="admin"
PASSWORD="yourpassword"
PASSWORD_MD5="$(printf '%s' "$PASSWORD" | openssl md5 | awk '{print $2}')"

curl -i "https://kosync.example.com/users/create" \
  -H "Accept: application/vnd.koreader.v1+json" \
  -H "Content-Type: application/json" \
  --data "{\"username\":\"$USERNAME\",\"password\":\"$PASSWORD_MD5\"}"
```

### Validate credentials

`x-auth-key` is the MD5 hash of the password.

```bash
curl "https://kosync.example.com/users/auth" \
  -H "Accept: application/vnd.koreader.v1+json" \
  -H "x-auth-user: admin" \
  -H "x-auth-key: <md5-of-password>"
```

### KOReader device setup

In KOReader: **Settings → Progress sync** → set server to `https://kosync.example.com`, enter username and MD5 password.

## Without Istio

Disable VirtualService and expose via NodePort or LoadBalancer:

```yaml
virtualService:
  enabled: false
service:
  type: LoadBalancer
  port: 17200
```

## Upgrade

```bash
helm upgrade kosync ./charts/kosync
```

## Uninstall

```bash
helm uninstall kosync
```

PVCs are not deleted automatically. Remove manually if no longer needed:

```bash
kubectl delete pvc -l app.kubernetes.io/instance=kosync
```
