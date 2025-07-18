# TeamSpeak 3 Server

This Helm chart deploys a TeamSpeak 3 server in your Kubernetes cluster with persistent storage and NodePort exposure.

## Features

- TeamSpeak 3 Server (latest version)
- Official Docker Hub image (teamspeak:latest)
- Persistent data storage for server configuration and virtual servers
- NodePort service for direct node access
- Configurable resource limits and requests
- Health checks for reliability

## Deployment

The TeamSpeak server will be automatically deployed by ArgoCD once this folder is committed to the repository.

## Connection Details

### Voice Connection
- **Server Address**: `<any-node-ip>:30987`
- **Protocol**: UDP
- **Default Port**: 30987

### Server Query (Admin)
- **Address**: `<any-node-ip>:30011`
- **Protocol**: TCP
- **Port**: 30011

### File Transfer
- **Address**: `<any-node-ip>:30033`
- **Protocol**: TCP
- **Port**: 30033

## Getting Node IPs

To find your cluster node IPs, run:

```bash
kubectl get nodes -o wide
```

Look for the `EXTERNAL-IP` or `INTERNAL-IP` columns.

## First-Time Setup

After deployment, the TeamSpeak server will generate initial credentials. To retrieve them:

1. Check the pod logs for the initial admin credentials:
   ```bash
   kubectl logs deployment/teamspeak -n default
   ```

2. Look for output similar to:
   ```
   ------------------------------------------------------------------
                         I M P O R T A N T                           
   ------------------------------------------------------------------
   Server Query Admin Account created
   loginname= "serveradmin", password= "XXXXXXXXX"
   ------------------------------------------------------------------
   ```

3. Use these credentials to connect via Server Query or TeamSpeak client with admin privileges.

## Configuration

### Environment Variables

The deployment accepts several environment variables in `values.yaml`:

- `TS3SERVER_LICENSE`: Set to "accept" to accept the license (required)
- `TS3_UID`: User ID for the TeamSpeak process (default: 1000)

### Persistence

By default, persistence is enabled with 1Gi storage. The data is mounted at `/var/ts3server` and includes:
- Server configuration
- Virtual server data
- User permissions
- Logs

### Resources

Default resource allocation:
- **Requests**: 128Mi RAM, 100m CPU
- **Limits**: 256Mi RAM, 200m CPU

## Accessing via Cloudflared (Optional)

If you want to expose the TeamSpeak server admin interface through your domain, you can add an entry to the Cloudflared configuration:

1. Edit `platform/cloudflared/config.yaml`
2. Add a new ingress entry:
   ```yaml
   - hostname: teamspeak.k8s.stormix.dev
     service: http://teamspeak.default.svc:10011
   ```
3. Commit and push the changes

Note: This only exposes the query interface. Voice connections will still need to use the NodePort.

## Security Considerations

- The server runs as non-root user (UID/GID 1000)
- Uses security contexts for enhanced pod security
- Network access is controlled via Kubernetes NetworkPolicies (if configured)

## Troubleshooting

### Server Not Starting
1. Check pod logs: `kubectl logs deployment/teamspeak`
2. Verify persistent volume is available
3. Check resource availability on nodes

### Connection Issues
1. Verify NodePort services are accessible: `kubectl get svc teamspeak`
2. Check firewall rules on nodes
3. Ensure ports 30987 (UDP), 30011 (TCP), and 30033 (TCP) are open

### Data Persistence Issues
1. Check PVC status: `kubectl get pvc`
2. Verify storage class is available
3. Check node disk space

## Scaling Considerations

TeamSpeak 3 server is designed to run as a single instance. Do not increase `replicaCount` beyond 1 as it may cause data corruption due to shared storage access.

## License

TeamSpeak 3 server requires accepting their license agreement. This is automatically done via the `TS3SERVER_LICENSE=accept` environment variable.

For production use, consider purchasing a proper TeamSpeak license from TeamSpeak Systems GmbH. 