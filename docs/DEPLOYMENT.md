# Deployment Guide

Fork-to-running deployment of the Butler observability pipeline reference.

## Prerequisites

- **Pipeline cluster.** A Kubernetes cluster designated as your observability pipeline. This is typically a Butler-managed tenant cluster, but any cluster with Flux and a storage class will work.

- **Flux v2.** Installed on the pipeline cluster, or you'll bootstrap it during deployment. Flux reconciles this repository's manifests onto the cluster.

- **StorageClass.** The reference defaults to `longhorn`. If your cluster uses a different storage class, update the `persistence.storageClassName` in the overlay values and the `storageClassName` in the cluster infrastructure.yaml patches.

- **Network reachability to backends.** The aggregator needs to reach your VictoriaMetrics, Loki, and Tempo instances over the network. Confirm connectivity from the pipeline cluster to each backend endpoint before deploying.

- **Network reachability from tenants.** Tenant clusters running Butler agents need to reach the aggregator's LoadBalancer IP. The LoadBalancer exposes ports 8080 (HTTP logs), 9000 (Prometheus remote write), 4317 (OTLP gRPC), and 4318 (OTLP HTTP). Confirm that tenant cluster networks can route to the pipeline cluster's LoadBalancer.

## Pre-Deployment Checklist

Six decisions to make before deploying. The first five are endpoint values to fill in. The sixth is an auth decision.

### 1. REQUIRED_VICTORIA_METRICS_ENDPOINT

Your VictoriaMetrics vminsert remote write URL. Must include the insert API route.

Example: `http://10.0.0.1:8480/insert/0/prometheus/api/v1/write`

This receives all metrics: the aggregator's own self-monitoring metrics and all Prometheus remote-write data forwarded from tenant clusters.

### 2. REQUIRED_LOKI_ENDPOINT

Your Loki push API base URL.

Example: `http://10.0.0.2:3100`

This receives all logs: application logs from tenant Vector agents (via the HTTP source) and the aggregator's own self-logs.

### 3. REQUIRED_TENANT_ID

Loki tenant identifier. Used in the `X-Scope-OrgID` header for all log pushes.

If your Loki runs in single-tenant mode (auth_enabled: false), use `default` or any consistent string. If multi-tenant, use the tenant ID assigned to your organization.

### 4. REQUIRED_TEMPO_ENDPOINT

Your Tempo OTLP HTTP endpoint for trace ingestion.

Example: `http://10.0.0.3:4318/v1/traces`

This receives OTLP traces forwarded from tenant clusters via the aggregator's OpenTelemetry source.

### 5. REQUIRED_PIPELINE_NAME

An identifier for this pipeline cluster, used as a static label on all Loki sinks. Helps distinguish logs from multiple pipeline clusters in a shared Loki instance.

Example: `obs-pipeline` or `my-org-pipeline`

### 6. Authentication Decision

The reference ships with no authentication — all sink endpoints use plain HTTP. This is the proven pattern when backends are on the same internal network as the aggregator.

If any of your backends require authentication:

1. Decide on a secret management approach (SealedSecrets, external-secrets-operator, or manual `kubectl create secret`)
2. Provision the required credentials as a Secret in the `vector` namespace on the pipeline cluster
3. Uncomment the auth blocks and env var mounts in the overlay values files

See [CUSTOMIZATION.md](CUSTOMIZATION.md) for detailed auth patterns per backend type.

## Step-by-Step Deployment

### 1. Fork the repository

Fork this repository to your Git hosting platform (GitHub, GitLab, etc.). All subsequent changes happen in your fork.

### 2. Rename cluster directories

Rename `clusters/pipeline-dev` and `clusters/pipeline-prd` to match your actual cluster names. Update the `externalLabels.cluster` value in each cluster's `infrastructure.yaml` to match.

For a single-environment deployment, delete the unused cluster directory and its corresponding apps overlay.

### 3. Replace placeholder values

In `apps/dev/vector-values.yaml` and `apps/prd/vector-values.yaml`, search for `REQUIRED_` and replace each placeholder with your actual values. There are 5 distinct placeholders, some appearing in multiple locations (the Loki endpoint and tenant ID appear in the main log sink and in commented-out optional sinks).

### 4. Adjust sizing if needed

The dev and prd overlays ship with production-proven sizing presets. Review them against your expected ingest volume. See [CUSTOMIZATION.md](CUSTOMIZATION.md) for scaling guidance.

### 5. Configure authentication (if needed)

If any backend requires auth, uncomment the relevant auth blocks in the overlay values and provision the referenced Secret on the cluster. See the Pre-Deployment Checklist above and [CUSTOMIZATION.md](CUSTOMIZATION.md) for patterns.

### 6. Bootstrap Flux

Bootstrap Flux on the pipeline cluster, pointing at your fork. Replace the placeholders in the bootstrap command with your repository details:

```sh
flux bootstrap github \
  --owner=YOUR_ORG \
  --repository=YOUR_REPO \
  --branch=main \
  --path=clusters/YOUR_CLUSTER_NAME \
  --personal
```

For GitLab, use `flux bootstrap gitlab` with equivalent flags.

Flux bootstrap installs Flux on the cluster, creates the GitRepository and root Kustomization, and overwrites the `flux-system/gotk-components.yaml` and `gotk-sync.yaml` placeholder files with real manifests.

### 7. Watch reconciliation

After bootstrap, Flux reconciles in tier order: infrastructure controllers first (kube-prometheus-stack), then apps (Vector) after infrastructure reports Ready.

```sh
# Watch Flux reconciliation
flux get all -w

# Watch HelmRelease status
kubectl get helmrelease -A -w

# Check kube-prometheus-stack (infrastructure tier)
kubectl get pods -n observability

# Check Vector (apps tier, deploys after infrastructure is Ready)
kubectl get pods -n vector
kubectl get hpa -n vector
```

Expected progression:
1. `infra-controllers` Kustomization reconciles, `kube-prometheus-stack` HelmRelease installs
2. Prometheus, kube-state-metrics, node-exporter pods become Ready in the `observability` namespace
3. `apps` Kustomization reconciles (depends on infra), Vector HelmRelease installs
4. Vector pods become Ready in the `vector` namespace, HPA active

### 8. Validate the deployment

Confirm the aggregator is running and reachable:

```sh
# Vector pods running at HPA minimum
kubectl get pods -n vector

# LoadBalancer IP allocated
kubectl get svc -n vector vector

# HelmRelease succeeded
kubectl get helmrelease -n vector vector
```

## Post-Deployment Validation

### Confirm data path

Send synthetic data to each receiver port to confirm end-to-end delivery.

**HTTP logs (port 8080):**

```sh
LB_IP=$(kubectl get svc -n vector vector -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

curl -X POST "http://${LB_IP}:8080" \
  -H "Content-Type: application/json" \
  -d '{"message": "{\"test\": true, \"source\": \"deployment-validation\"}"}'
```

Confirm the event arrives in Loki by querying for `{service_name="vector"}` within the last 5 minutes.

**Prometheus remote write (port 9000):**

If a Prometheus instance is configured to remote-write to this aggregator, check VictoriaMetrics for metrics with the `cluster` external label matching the source Prometheus.

**OTLP traces (port 4317/4318):**

If an application sends OTLP traces, check Tempo for traces from the expected service.

### Confirm self-monitoring

The aggregator's own metrics flow through the pipeline: Vector internal_metrics → victoria_metrics sink → your VictoriaMetrics instance. Query for:

```promql
# Events processed per second by component
rate(vector_pipeline_self_metrics_component_sent_events_total[5m])

# Buffer utilization
vector_pipeline_self_metrics_buffer_byte_size

# Errors by component
rate(vector_pipeline_self_metrics_component_errors_total[5m])
```

## Troubleshooting

### LoadBalancer stuck in Pending

The Vector service requests a LoadBalancer. If no LoadBalancer controller is installed (MetalLB, cloud provider LB controller, or similar), the service stays Pending indefinitely. Check:

```sh
kubectl describe svc -n vector vector
```

Look for events indicating no IP allocator is available.

### Vector pods OOMKilled

If ingest volume exceeds the memory limit, Vector pods get OOMKilled. Indicators:

```sh
kubectl get pods -n vector --field-selector=status.phase!=Running
kubectl describe pod -n vector <pod-name> | grep -A5 "Last State"
```

Fix: increase `resources.limits.memory` in the overlay values. Also check `buffer.max_size` — disk buffers absorb traffic spikes, but if disk fills before the sink drains, Vector applies backpressure which increases memory usage on the ingest path.

### Sink delivery failures

Vector logs delivery failures to its own internal_logs, which flow to the loki_logs sink. If the Loki sink itself is failing, check Vector pod logs directly:

```sh
kubectl logs -n vector -l app.kubernetes.io/instance=vector --tail=100
```

Common causes: backend unreachable (firewall, DNS, wrong IP), authentication rejected (wrong credentials or expired token), backend overloaded (429 responses).

### HPA not scaling

HPA requires the metrics-server to be installed on the cluster. Check:

```sh
kubectl get apiservice v1beta1.metrics.k8s.io
kubectl top pods -n vector
```

If metrics-server is missing, HPA stays at minReplicas regardless of load.

### Infrastructure tier stuck

If the `infra-controllers` Kustomization fails, the `apps` Kustomization won't start (it depends on infra). Check:

```sh
flux get kustomizations
kubectl get helmrelease -n observability kube-prometheus-stack
```

Common causes: CRD installation failures (cluster version too old), resource limits too tight for the cluster's capacity, webhook timeouts during install (the reference disables admission webhooks to avoid this).
