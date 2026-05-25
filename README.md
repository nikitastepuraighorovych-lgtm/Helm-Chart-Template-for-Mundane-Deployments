# Helm Chart Template for Mundane Deployments

This is a simple Helm chart I made for myself so it would be relatively easy to use and rewrite for my needs.

---

## Structure

```
template-helm/
├── README.md
└── template/                   # The Helm chart
    ├── Chart.yaml
    ├── values.yaml
    ├── charts/
    ├── config/                 # Files for ConfigMap glob
    ├── secrets/                # Files for Secret glob
    └── templates/
        ├── _helpers.tpl
        ├── deployment.yaml
        ├── service.yaml
        ├── configmap.yaml
        ├── secrets.yaml
        ├── pvc.yaml
        ├── ingress.yaml
        ├── hpa.yaml
        ├── job.yaml
        ├── cronjob.yaml
        └── tests/
            └── test-connection.yaml
```

---

## Features

| Feature | Toggle | Description |
|---------|--------|-------------|
| Deployment | Always on | Primary container with configurable command, args, env |
| Init Containers | `initContainers: []` | Run setup tasks before main container |
| Sidecar Containers | `sidecarContainers: []` | Run alongside main container |
| Service | Always on | ClusterIP / NodePort / LoadBalancer |
| Ingress | `ingress.enabled` | HTTP/HTTPS routing |
| ConfigMap | `configMap.enabled` | Inline data, files, or directory glob |
| Secret | `secret.enabled` | Inline data, files, or directory glob |
| PVC | `persistence.enabled` | Persistent storage |
| Shared Volume | `sharedVolume.enabled` | emptyDir between containers |
| Job | `job.enabled` | One-time task (migrations, setup) |
| CronJobs | `cronJobs: []` | Scheduled recurring tasks |
| HPA | `autoscaling.enabled` | Horizontal Pod Autoscaler |
| Tests | `tests.enabled` | Helm test connection |

---

## Quick Start

### Install

```bash
helm install my-release ./template
```

### Install with custom values

```bash
helm install my-release ./template -f my-values.yaml
```

### Dry run (preview what gets created)

```bash
helm template my-release ./template
helm install my-release ./template --dry-run --debug
```

### Upgrade

```bash
helm upgrade my-release ./template -f my-values.yaml
```

### Uninstall

```bash
helm uninstall my-release
```

---

## Configuration

### Minimal values.yaml (busybox -- just runs)

```yaml
replicaCount: 1

image:
  repository: busybox
  tag: "1.36"
  pullPolicy: IfNotPresent

command:
  - sh
  - -c
  - "echo 'Running' && sleep infinity"

service:
  type: ClusterIP
  port: 80

livenessProbe: {}
readinessProbe: {}
```

### Nginx example (with probes)

```yaml
replicaCount: 2

image:
  repository: nginx
  tag: "1.27-alpine"
  pullPolicy: IfNotPresent

command: []

service:
  type: ClusterIP
  port: 80

livenessProbe:
  httpGet:
    path: /
    port: http

readinessProbe:
  httpGet:
    path: /
    port: http

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi
```

---

## Feature Examples

### ConfigMap -- inline data

```yaml
configMap:
  enabled: true
  data:
    APP_ENV: "production"
    LOG_LEVEL: "info"
  files: []
  filesGlob: ""
```

### ConfigMap -- from files in directory

Place files in `template/config/` then:

```yaml
configMap:
  enabled: true
  data: {}
  files: []
  filesGlob: "config/*"
```

### Secret -- inline

```yaml
secret:
  enabled: true
  type: Opaque
  stringData:
    DB_PASSWORD: "changeme"
    API_KEY: "changeme"
  data: {}
  files: []
  filesGlob: ""
```

### Secret -- from files in directory

Place files in `template/secrets/` then:

```yaml
secret:
  enabled: true
  type: Opaque
  stringData: {}
  data: {}
  files: []
  filesGlob: "secrets/*"
```

**Warning:** Don't commit real secrets to Git. Use `--set` flag, sealed-secrets, external-secrets, or Vault instead.

### Persistence (PVC)

```yaml
persistence:
  enabled: true
  accessMode: ReadWriteOnce
  size: 5Gi
  mountPath: /data
  # storageClass: "gp3"
```

### Init Containers

```yaml
initContainers:
  - name: init-permissions
    image:
      repository: busybox
      tag: "1.36"
      pullPolicy: IfNotPresent
    command:
      - sh
      - -c
      - |
        echo "Setting up..."
        mkdir -p /data/app
        chmod -R 777 /data/app
    volumeMounts:
      - name: data
        mountPath: /data
    resources:
      limits:
        cpu: 100m
        memory: 64Mi
```

### Sidecar Containers

```yaml
sidecarContainers:
  - name: log-shipper
    image:
      repository: busybox
      tag: "1.36"
      pullPolicy: IfNotPresent
    command:
      - sh
      - -c
      - "tail -F /data/logs/*.log 2>/dev/null || sleep infinity"
    volumeMounts:
      - name: data
        mountPath: /data
    resources:
      limits:
        cpu: 100m
        memory: 64Mi
```

### Job (one-time task)

```yaml
job:
  enabled: true
  image:
    repository: busybox
    tag: "1.36"
    pullPolicy: IfNotPresent
  command:
    - sh
    - -c
    - |
      echo "Running migration..."
      sleep 5
      echo "Done!"
  backoffLimit: 3
  restartPolicy: Never
  # Run as Helm hook:
  # annotations:
  #   "helm.sh/hook": post-install
  #   "helm.sh/hook-delete-policy": hook-succeeded
```

### CronJobs (scheduled tasks)

```yaml
cronJobs:
  - name: db-backup
    enabled: true
    schedule: "0 2 * * *"
    image:
      repository: busybox
      tag: "1.36"
      pullPolicy: IfNotPresent
    command:
      - sh
      - -c
      - |
        echo "Backup started at $(date)"
        echo "Backup done."
    concurrencyPolicy: Forbid
    backoffLimit: 3
    successfulJobsHistoryLimit: 3
    failedJobsHistoryLimit: 3
    restartPolicy: Never
    resources:
      limits:
        cpu: 200m
        memory: 256Mi
```

### Ingress

```yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: app-tls
      hosts:
        - app.example.com
```

### Autoscaling (HPA)

```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
```

### Helm Tests

```yaml
tests:
  enabled: true
  image:
    repository: busybox
    tag: "1.36"
```

```bash
helm test my-release
```

---

## Where Things Mount Inside the Pod

```
+----------------------------------------------+
|                    POD                        |
|                                               |
|  /data/           <- PVC (persistent)         |
|  /shared/         <- emptyDir (temporary)     |
|  /config/     <- ConfigMap (read-only)        |
|  /secrets/    <- Secret (read-only, 0400)     |
|                                               |
|  ENV VARS:                                    |
|    From configMap.data (via envFrom)          |
|    From secret.stringData (via envFrom)       |
|    From env[] (inline)                        |
+----------------------------------------------+
```

---

## Cron Schedule Cheat Sheet

```
# +-------------- minute (0-59)
# | +------------ hour (0-23)
# | | +---------- day of month (1-31)
# | | | +-------- month (1-12)
# | | | | +------ day of week (0-6, Sun=0)
# | | | | |
# * * * * *

"*/15 * * * *"      # Every 15 minutes
"0 * * * *"         # Every hour
"0 2 * * *"         # Every day at 2:00 AM
"0 0 * * 0"         # Every Sunday at midnight
"0 9 * * 1-5"       # Weekdays at 9:00 AM
"0 */6 * * *"       # Every 6 hours
"30 4 1 * *"        # 4:30 AM on 1st of each month
```

---

## Useful Commands

```bash
# Preview rendered templates
helm template my-release ./template

# Install
helm install my-release ./template

# Install with overrides
helm install my-release ./template \
  --set image.repository=nginx \
  --set image.tag=1.27-alpine \
  --set command='' \
  --set persistence.enabled=true

# Upgrade
helm upgrade my-release ./template -f production-values.yaml

# Rollback
helm rollback my-release 1

# Check status
helm status my-release
kubectl get pods,svc,configmap,secret,pvc,jobs,cronjobs

# Logs
kubectl logs deploy/my-release-template
kubectl logs job/my-release-template

# Uninstall
helm uninstall my-release
```

---

## License

Do whatever you want with it.

---

## Author

Nikita Stepura -- nikitastepuraighorovych@gmail.com
