# Temporal Deployment Guide

This guide covers every issue encountered getting Temporal running, the root cause and fix
for each, how to deploy on a new cluster, how to access Temporal, and troubleshooting steps.

---

## What Broke and How It Was Fixed

### Fix 1 — Cassandra OOMKilled (Exit Code 137)

**Symptom:** Cassandra pod crashed within seconds of startup, always dying silently right
after logging JVM arguments. Exit code 137 = OOM kill.

**Root cause:** Cassandra 3.11.x uses the `sigar` native library which reads **host** system
RAM (e.g. 16 GB) instead of the container's cgroup limit (2 Gi). On a 16-CPU host jemalloc
creates 64 memory arenas (4 × ncpus), each several MB in size. Total native allocation
exceeds the 2 Gi container limit within seconds; the kernel OOM killer fires.

**Fix:**
- Upgraded to `cassandra:4.0`. This version replaced `sigar` with `OSHI`, which is
  container-aware and reads cgroup limits. Java 11 `UseContainerSupport` is also enabled
  by default, so the JVM respects the container memory limit.
- Set `MALLOC_CONF: "narenas:4,background_thread:false"` to hard-cap jemalloc arenas at 4
  regardless of CPU count.
- Increased heap to `max_heap_size: 512M` (from 256M). At 256M, GC pressure was so severe
  that the liveness probe (`nodetool status` via JMX, 5s timeout) kept timing out and
  killing the pod before it ever became Ready.
- Increased probe `timeoutSeconds: 30` and `initialDelaySeconds: 120` to give Cassandra
  enough startup time.

---

### Fix 2 — StatefulSet Not Rolling Out Config Changes

**Symptom:** Config changes were pushed to git and ArgoCD reported Synced, but the
Cassandra pod kept running with the old spec.

**Root cause:** The Cassandra subchart defaults to `updateStrategy: OnDelete`. ArgoCD
applied the new StatefulSet spec to Kubernetes, but Kubernetes only applies `OnDelete`
changes when the pod is manually deleted.

**Fix:** Added `updateStrategy: type: RollingUpdate` to `values.yaml`.

---

### Fix 3 — Visibility Store Empty (All Server Pods CrashLoop)

**Symptom:** All four Temporal server pods (`frontend`, `history`, `matching`, `worker`)
crashed immediately with:
```
config validation error: persistence config error: datastore "visibility":
must provide config for one and only one datastore: elasticsearch, cassandra, sql or custom store
```

**Root cause:** A design gap in the Temporal Helm chart (v0.48.0). The configmap template
only has branches for Elasticsearch and SQL under the visibility section:

```yaml
visibility:
{{- if or $elasticsearch.enabled $elasticsearch.external }}
  elasticsearch: ...
{{- else if eq (include "temporal.persistence.driver" ...) "sql" }}
  sql: ...
{{- end }}
```

There is **no Cassandra branch**. The chart's default `values.yaml` sets
`visibility.driver: "cassandra"`, but nothing is rendered for it. The visibility section
is left empty. Temporal refuses to start without a configured visibility store.

**Fix:**
- Switched visibility to PostgreSQL (`postgres12` SQL driver), which is supported by the
  chart template.
- Deployed a minimal PostgreSQL pod (`temporal-visibility-postgresql`) via a raw Kubernetes
  manifest in `temporal/manifests/postgres-visibility.yaml`.
- Added `temporal/manifests` as a third ArgoCD source so the PostgreSQL resources are
  deployed alongside the Helm chart.

---

### Fix 4 — ArgoCD SyncError: Job spec.template Is Immutable

**Symptom:** After changing the Cassandra image tag (`3.11.3` → `4.0`), ArgoCD showed:
```
Job.batch "temporal-schema-1" is invalid: spec.template: ... field is immutable
```

**Root cause:** The Cassandra image tag change propagated into the schema Job's
`check-cassandra` init container image. Kubernetes Jobs have an immutable `spec.template`
after creation — ArgoCD cannot patch it.

**Fix:** Added `ignoreDifferences` in `app.yaml` to prevent ArgoCD from ever trying to diff
or patch a Job's spec.template:

```yaml
ignoreDifferences:
  - group: batch
    kind: Job
    jqPathExpressions:
      - .spec.template
```

When the schema Job needs to be recreated (e.g. after a Cassandra image change), delete it
manually and ArgoCD recreates it fresh on next sync.

---

### Fix 5 — app.yaml Changes Not Self-Applying

**Symptom:** Structural changes to `temporal/app.yaml` (adding a source, ignoreDifferences,
etc.) pushed to git had no effect on the cluster.

**Root cause:** `app.yaml` defines the ArgoCD Application resource. There is no
"app of apps" managing it — it must be applied manually. ArgoCD cannot manage itself.
Only *content* changes (values.yaml, raw manifests) are picked up automatically.

**Fix:** Apply manually whenever the structure of `app.yaml` changes:
```bash
kubectl apply -f temporal/app.yaml
```

> Regular changes to `values.yaml` or files in `temporal/manifests/` do **not** require
> re-applying `app.yaml` — ArgoCD picks those up automatically.

---

### Fix 6 — Constant gocql Session Refresh Errors (Hundreds Per Second)

**Symptom:** All server pods were `1/1 Running` and serving requests, but the logs showed
hundreds of errors per second:
```
gocql wrapper: unable to refresh gocql session — no connections were made when creating the session
```

**Root cause:** A bug in the Temporal Helm chart's `cassandra.hosts` helper template. The
template loops over `cluster_size` nodes and appends a trailing comma after every hostname:

```
{{- printf "%s.%s," $cassandraName $.Release.Namespace -}}
```

With `cluster_size: 1` this produces `temporal-cassandra.temporal,` — note the trailing
comma. When the Temporal server reads this config and passes the hosts to gocql, gocql
splits on commas and gets two entries: `temporal-cassandra.temporal` (valid) and `""`
(empty string, invalid). Every background session refresh fails trying to dial the empty
host.

**Fix:** Explicitly set `server.config.persistence.default.cassandra.hosts` in `values.yaml`
to bypass the buggy template:

```yaml
server:
  config:
    persistence:
      default:
        cassandra:
          hosts:
            - "temporal-cassandra"
```

With an explicit list, the template uses `join ","` on a single-element list, producing
`temporal-cassandra` — no trailing comma.

---

## File Structure

```
temporal/
├── app.yaml                        # ArgoCD Application definition (apply manually)
├── values.yaml                     # Helm chart overrides
├── manifests/
│   └── postgres-visibility.yaml   # PostgreSQL for visibility store
└── GUIDE.md                        # This file
```

---

## Deploying on a New Cluster

### Prerequisites

- Kubernetes cluster with at least one worker node
- ArgoCD installed in the `argocd` namespace
- Network access from the cluster to `github.com` and `go.temporal.io`
- At least **4 Gi free memory** across the cluster (Cassandra 2 Gi + PostgreSQL 512 Mi +
  Temporal server pods ~1 Gi)

---

### Step 1 — (Optional) Pin Cassandra to a specific node

By default Kubernetes schedules Cassandra freely. If one of your nodes is overcommitted,
uncomment and set the nodeSelector in `values.yaml`:

```yaml
cassandra:
  # selector:
  #   nodeSelector:
  #     kubernetes.io/hostname: <your-node-name>
```

Find the least-loaded node:
```bash
kubectl describe nodes | grep -A6 "Allocated resources"
```

---

### Step 2 — Apply the ArgoCD Application

```bash
kubectl apply -f temporal/app.yaml
```

ArgoCD will immediately begin syncing all three sources:
1. The Temporal Helm chart from `go.temporal.io/helm-charts`
2. `values.yaml` from this git repo
3. Raw manifests from `temporal/manifests/` (PostgreSQL)

---

### Step 3 — Wait for Cassandra to become Ready

Cassandra takes 2–3 minutes to start (the readiness probe has a 120s initial delay).

```bash
kubectl get pods -n temporal -w
# Wait until temporal-cassandra-0 shows 1/1 Running
```

---

### Step 4 — Wait for the Schema Job to Complete

After Cassandra is Ready, ArgoCD runs the schema Job which:
1. Creates the `temporal` keyspace in Cassandra
2. Creates the `temporal_visibility` database in PostgreSQL
3. Runs all schema migrations for both stores

```bash
kubectl get job -n temporal -w
# Wait until temporal-schema-1 shows COMPLETIONS: 1/1
```

If the Job fails or gets stuck, delete and re-sync:
```bash
kubectl delete job temporal-schema-1 -n temporal
kubectl -n argocd patch application temporal --type merge \
  -p '{"operation":{"sync":{"syncStrategy":{"hook":{"force":true}}}}}'
```

---

### Step 5 — Verify all pods are Running

```bash
kubectl get pods -n temporal
```

Expected healthy state:
```
NAME                                READY   STATUS
temporal-admintools-xxx             1/1     Running
temporal-cassandra-0                1/1     Running
temporal-frontend-xxx               1/1     Running
temporal-history-xxx                1/1     Running
temporal-matching-xxx               1/1     Running
temporal-schema-1-xxx               0/1     Completed
temporal-visibility-postgresql-xxx  1/1     Running
temporal-web-xxx                    1/1     Running
temporal-worker-xxx                 1/1     Running
```

---

## How to Access Temporal

### Web UI

The Temporal Web UI (`temporal-web`) runs on port 8088. It lets you browse namespaces,
search workflows, view execution history, and inspect task queues.

**Port-forward (quickest):**
```bash
kubectl port-forward -n temporal svc/temporal-web 8088:8088
```
Then open: **http://localhost:8088**

**Check what's available:**
```
http://localhost:8088          — Namespace list
http://localhost:8088/namespaces/default/workflows  — Workflow list
```

---

### tctl CLI (Admin Tool)

`tctl` is the Temporal command-line tool. It is pre-installed in the `temporal-admintools`
pod.

**Open a shell:**
```bash
kubectl exec -it -n temporal deployment/temporal-admintools -- bash
```

**Or run a single command:**
```bash
kubectl exec -n temporal deployment/temporal-admintools -- tctl <command>
```

**Useful commands:**
```bash
# Check cluster health
tctl cluster health

# List namespaces
tctl namespace list

# Create a namespace
tctl namespace register --namespace my-app

# Describe a namespace
tctl namespace describe --namespace my-app

# List running workflows in a namespace
tctl --namespace my-app workflow list

# Show details of a workflow
tctl --namespace my-app workflow show --workflow_id <id> --run_id <run-id>

# Terminate a stuck workflow
tctl --namespace my-app workflow terminate --workflow_id <id> --reason "manual"
```

---

### gRPC API (Frontend Service)

The Temporal gRPC API is exposed by `temporal-frontend` on port **7233**.
This is what SDKs and `tctl` connect to.

**Port-forward:**
```bash
kubectl port-forward -n temporal svc/temporal-frontend 7233:7233
```

**From inside the cluster:** use `temporal-frontend.temporal:7233`

---

### Connecting an SDK

Use the gRPC frontend address when building workflows with the Temporal SDK.

**Go SDK:**
```go
import "go.temporal.io/sdk/client"

c, err := client.Dial(client.Options{
    HostPort:  "temporal-frontend.temporal:7233",  // inside cluster
    Namespace: "default",
})
```

**Python SDK:**
```python
from temporalio.client import Client

client = await Client.connect(
    "temporal-frontend.temporal:7233",  # inside cluster
    namespace="default",
)
```

**Java SDK:**
```java
WorkflowServiceStubs service = WorkflowServiceStubs.newLocalServiceStubs(
    WorkflowServiceStubsOptions.newBuilder()
        .setTarget("temporal-frontend.temporal:7233")
        .build()
);
```

If connecting **from outside the cluster**, port-forward first:
```bash
kubectl port-forward -n temporal svc/temporal-frontend 7233:7233
# then use "localhost:7233" as the target
```

---

### HTTP API (Frontend Service)

Temporal 1.25+ also exposes an HTTP/JSON API on port **7243**:

```bash
kubectl port-forward -n temporal svc/temporal-frontend 7243:7243
```

Check cluster health:
```bash
curl http://localhost:7243/api/v1/namespaces
```

---

## Troubleshooting

### Cassandra keeps crashing (OOMKilled)

Check which node it landed on and how much memory is allocated:
```bash
kubectl get pod temporal-cassandra-0 -n temporal -o wide
kubectl describe node <node-name> | grep -A6 "Allocated resources"
```

If the node is overcommitted, pin Cassandra to a less busy node by uncommenting
`cassandra.selector.nodeSelector` in `values.yaml`.

---

### Server pods crash: "must provide config for one and only one datastore"

The visibility store is empty in the generated config. Verify `values.yaml` has:
```yaml
server:
  config:
    persistence:
      visibility:
        driver: "sql"
        sql:
          driver: "postgres12"
          host: "temporal-visibility-postgresql"
          port: 5432
          database: "temporal_visibility"
          user: "temporal"
          password: "temporal"
```

Also check the PostgreSQL pod is running:
```bash
kubectl get pods -n temporal | grep postgresql
```

---

### Constant "gocql: no connections were made" errors in logs

The hosts string in the configmap has a trailing comma. Verify `values.yaml` has the
explicit hosts list:
```yaml
server:
  config:
    persistence:
      default:
        cassandra:
          hosts:
            - "temporal-cassandra"
```

Check the live configmap to confirm:
```bash
kubectl get configmap temporal-config -n temporal \
  -o jsonpath='{.data.config_template\.yaml}' | grep "hosts:"
# Should show: hosts: "temporal-cassandra"   (no trailing comma)
```

---

### ArgoCD SyncError: Job spec.template is immutable

The schema Job spec changed (e.g. Cassandra image tag updated). Delete the Job and sync:
```bash
kubectl delete job temporal-schema-1 -n temporal
kubectl -n argocd patch application temporal --type merge \
  -p '{"operation":{"sync":{"syncStrategy":{"hook":{"force":true}}}}}'
```

---

### Server pods crash: "unable to read DB schema version"

The server started before the schema Job finished setting up PostgreSQL. Wait for the
schema Job to complete (`COMPLETIONS: 1/1`), then restart the crashing pods:
```bash
kubectl delete pod -n temporal -l app.kubernetes.io/name=temporal
```

---

### Changes to app.yaml have no effect

`app.yaml` must be applied manually — ArgoCD does not manage itself:
```bash
kubectl apply -f temporal/app.yaml
```

---

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    temporal namespace                     │
│                                                          │
│  ┌─────────────────┐    ┌────────────────────────────┐  │
│  │   Cassandra 4.0 │    │  PostgreSQL 14             │  │
│  │   default store │    │  visibility store          │  │
│  │   keyspace:     │    │  database:                 │  │
│  │     temporal    │    │    temporal_visibility     │  │
│  └────────┬────────┘    └──────────────┬─────────────┘  │
│           │  (gRPC/gocql)              │  (SQL/pgx)     │
│  ┌────────▼────────────────────────────▼─────────────┐  │
│  │           Temporal Server — 4 services             │  │
│  │                                                    │  │
│  │  frontend :7233/:7243  ◄── SDK / tctl / HTTP API  │  │
│  │  history  :7234        ◄── workflow state machine  │  │
│  │  matching :7235        ◄── task queue matching     │  │
│  │  worker   :6939        ◄── internal workflows      │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌──────────────┐   ┌──────────────────────────────┐    │
│  │  admintools  │   │  temporal-web (UI) :8088      │    │
│  │  (tctl CLI)  │   │  http://localhost:8088        │    │
│  └──────────────┘   └──────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

| Component | Purpose | Port |
|---|---|---|
| `temporal-frontend` | gRPC API for SDK/tctl | 7233 (gRPC), 7243 (HTTP) |
| `temporal-history` | Workflow state machine | 7234 |
| `temporal-matching` | Task queue management | 7235 |
| `temporal-worker` | Internal system workflows | 6939 |
| `temporal-web` | Browser UI | 8088 |
| `temporal-admintools` | `tctl` CLI pod | — |
| `temporal-cassandra-0` | Workflow state storage | 9042 |
| `temporal-visibility-postgresql` | Workflow search index | 5432 |

### Data stores

| Store | Engine | What it holds |
|---|---|---|
| Default | Cassandra | Workflow state, history events, task queues, timers |
| Visibility | PostgreSQL | Workflow search index (status, start time, workflow type, custom search attributes) |

The Temporal Helm chart v0.48.0 has no Cassandra template for the visibility store.
PostgreSQL is the lightweight alternative to Elasticsearch for dev/test deployments.
