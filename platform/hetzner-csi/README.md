# Hetzner Cloud CSI Driver

This directory contains the configuration for deploying the [Hetzner Cloud CSI Driver](https://github.com/hetznercloud/csi-driver) to provide persistent volumes for Kubernetes workloads.

## Features

- **ReadWriteOnce volumes**: Provides block storage volumes that can be mounted to one node at a time
- **Volume expansion**: Supports expanding volumes without downtime
- **Multiple storage classes**: Different configurations for various use cases
- **Monitoring integration**: Metrics available for Prometheus monitoring

## Prerequisites

- Hetzner Cloud API token with read/write permissions
- Kubernetes cluster running on Hetzner Cloud instances
- ArgoCD for GitOps deployment

## Setup

### 1. Create Hetzner Cloud API Token

1. Go to the [Hetzner Cloud Console](https://console.hetzner.cloud/)
2. Navigate to your project
3. Go to "Security" → "API Tokens"
4. Create a new token with read/write permissions
5. Save the token securely

### 2. Create Sealed Secret

Create a sealed secret containing your Hetzner Cloud API token:

```bash
# Create a regular secret first
kubectl create secret generic hcloud-csi \
  --from-literal=token=YOUR_HETZNER_API_TOKEN \
  --namespace=kube-system \
  --dry-run=client -o yaml | kubeseal -o yaml > hcloud-csi-secret-sealed.yaml

# Update the template with the encrypted data
```

Replace the `token` field in `templates/hcloud-csi-secret.yaml` with the encrypted value from kubeseal.

### 3. Deploy via ArgoCD

The Hetzner CSI driver will be automatically deployed via ArgoCD when you:

1. Ensure the sealed secret is properly configured
2. Commit and push your changes to the repository
3. ArgoCD will sync the `hetzner-csi` application

## Storage Classes

The following storage classes are available:

### hcloud-csi
- **Use case**: General purpose persistent volumes
- **Reclaim policy**: Delete
- **Volume expansion**: Enabled
- **File system**: ext4

### hcloud-csi-retain
- **Use case**: Data that should persist even after PVC deletion
- **Reclaim policy**: Retain
- **Volume expansion**: Enabled
- **File system**: ext4

## Usage Examples

### Basic PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: hcloud-csi
  resources:
    requests:
      storage: 10Gi
```

### StatefulSet with Hetzner Storage

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-app
spec:
  serviceName: my-app
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app:latest
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: hcloud-csi
      resources:
        requests:
          storage: 20Gi
```

## Monitoring

The CSI driver includes Prometheus metrics and ServiceMonitor configuration for monitoring:

- Volume provisioning metrics
- Volume attachment/detachment metrics
- Performance metrics

Metrics will be automatically discovered by the kube-prometheus-stack.

## Troubleshooting

### Check CSI Driver Status

```bash
# Check CSI driver pods
kubectl get pods -n kube-system -l app.kubernetes.io/name=hcloud-csi

# Check CSI driver logs
kubectl logs -n kube-system -l app.kubernetes.io/name=hcloud-csi-controller

# Check storage classes
kubectl get storageclass
```

### Verify API Token

```bash
# Check if the secret exists
kubectl get secret hcloud-csi -n kube-system

# Verify token is valid (from a pod with access)
kubectl run test-pod --image=curlimages/curl -it --rm -- sh
# Inside the pod:
# TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
# curl -H "Authorization: Bearer $TOKEN" https://api.hetzner.cloud/v1/servers
```

### Volume Issues

```bash
# Check PVC status
kubectl get pvc

# Check PV details
kubectl get pv

# Check events for troubleshooting
kubectl get events --sort-by=.metadata.creationTimestamp
```

## Limitations

- **ReadWriteOnce only**: Hetzner Cloud volumes can only be attached to one node at a time
- **Same datacenter**: Volumes can only be attached to nodes in the same Hetzner datacenter
- **Volume limits**: Check Hetzner Cloud limits for maximum number of volumes per server

## References

- [Hetzner Cloud CSI Driver Documentation](https://github.com/hetznercloud/csi-driver)
- [Kubernetes CSI Documentation](https://kubernetes-csi.github.io/docs/)
- [Hetzner Cloud API Documentation](https://docs.hetzner.cloud/) 