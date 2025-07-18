# MinIO Object Storage

This Helm chart deploys MinIO, a high-performance object storage server compatible with Amazon S3 API, in your Kubernetes cluster.

## Features

- MinIO latest stable release
- S3-compatible API 
- Web-based management console
- Persistent storage with configurable size
- Secure sealed secret management for credentials
- CloudFlare tunnel integration for external access
- Health checks and monitoring ready
- Kubernetes-native with StatefulSet deployment

## Access

### MinIO Console (Web UI)
- **URL**: https://minio.k8s.stormix.dev
- **Username**: From sealed secret (default: minioadmin)
- **Password**: From sealed secret (generated during setup)

### MinIO API (S3 Compatible)
- **Endpoint**: https://minio-api.k8s.stormix.dev
- **Access Key**: From sealed secret
- **Secret Key**: From sealed secret

## Configuration

The MinIO deployment is configured via `values.yaml`:

- **Storage**: 10Gi persistent volume (configurable)
- **Resources**: 256Mi-1Gi memory, 250m-500m CPU
- **Replicas**: 1 (single-node deployment)
- **Security**: Non-root container with dedicated user/group

## Credentials Management

Credentials are managed via sealed secrets in `applications/shared-secrets/minio.yaml`:

- `MINIO_ROOT_USER`: Administrator username
- `MINIO_ROOT_PASSWORD`: Administrator password

To update credentials, recreate the sealed secret:

```bash
kubectl create secret generic minio \
  --from-literal=MINIO_ROOT_USER=your-username \
  --from-literal=MINIO_ROOT_PASSWORD=your-secure-password \
  --namespace=default --dry-run=client -o yaml | kubeseal -o yaml > applications/shared-secrets/minio.yaml
```

## Storage

- **Default Size**: 10Gi
- **Storage Class**: Uses cluster default (hcloud-csi for Hetzner)
- **Access Mode**: ReadWriteOnce
- **Mount Path**: `/data`

## Health Checks

The deployment includes comprehensive health checks:

- **Liveness Probe**: `/minio/health/live` endpoint
- **Readiness Probe**: `/minio/health/ready` endpoint
- **Startup Delay**: 30s for liveness, 10s for readiness

## Deployment

MinIO will be automatically deployed by ArgoCD once this application is committed to the repository. The deployment includes:

1. StatefulSet for persistent MinIO instance
2. PersistentVolumeClaim for data storage
3. Services for API and console access
4. CloudFlare tunnel integration for external access

## Usage Examples

### AWS CLI Configuration

```bash
aws configure set aws_access_key_id your-minio-access-key
aws configure set aws_secret_access_key your-minio-secret-key
aws configure set default.region us-east-1
aws configure set default.s3.signature_version s3v4

# Test connection
aws --endpoint-url https://minio-api.k8s.stormix.dev s3 ls
```

### Python SDK (boto3)

```python
import boto3

client = boto3.client(
    's3',
    endpoint_url='https://minio-api.k8s.stormix.dev',
    aws_access_key_id='your-access-key',
    aws_secret_access_key='your-secret-key',
    region_name='us-east-1'
)

# List buckets
buckets = client.list_buckets()
```

### Creating Buckets

Using the web console:
1. Navigate to https://minio.k8s.stormix.dev
2. Login with your credentials
3. Click "Create Bucket"
4. Configure bucket name and settings

Using CLI:
```bash
aws --endpoint-url https://minio-api.k8s.stormix.dev s3 mb s3://my-bucket
```

## Monitoring

MinIO exposes Prometheus metrics on the `/minio/v2/metrics/cluster` endpoint. The metrics include:

- Storage usage and capacity
- Request rates and latencies  
- Node health and status
- Network and disk metrics

## Troubleshooting

### Check Pod Status
```bash
kubectl get pods -l app.kubernetes.io/name=minio
kubectl logs statefulset/minio
```

### Verify Storage
```bash
kubectl get pvc minio-data
kubectl describe pvc minio-data
```

### Check Service Connectivity
```bash
kubectl get svc minio minio-console
kubectl port-forward svc/minio-console 9001:9001
```

### CloudFlare Tunnel Status
```bash
kubectl logs deployment/cloudflared -n default
``` 