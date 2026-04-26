# Temporal Deployment Guide

This guide explains what went wrong with the initial Temporal deployment, what was fixed,
and how to deploy Temporal from scratch on a new cluster.

---

## What Broke and Why

### Problem 1 — Cassandra OOMKilled (Exit Code 137)

**Symptom:** Cassandra pod crashed immediately on startup, always dying silently after logging JVM arguments.

**Root cause:** Cassandra 3.11.x uses the `sigar` native library to read the *host* system memory (e.g. 16 GB) instead of the container's cgroup limit (2 Gi). On a host with 16 CPUs, jemalloc creates 64 arenas (4 × ncpus), each several MB, pushing native memory well past the 2 Gi container limit within seconds of startup. The OOM killer terminates the process.

**Fix:**
- Upgraded Cassandra to `4.0`. Version 4.0 replaced `sigar` with the `OSHI` library, which is container-aware and reads cgroup memory limits correctly. It also ships with Java 11 `UseContainerSupport` enabled by default.
- Set `MALLOC_CONF: "narenas:4,background_thread:false"` to cap jemalloc arenas at 4 regardless of CPU count.
- Increased heap to `max_heap_size: 512M` (from 256M). At 256M, GC pressure was so severe that the liveness probe (`nodetool status` via JMX) timed out at 5s, causing the pod to be killed and restarted in a loop before it ever became Ready.
- Increased probe `timeoutSeconds` from 5 to 30 and `initialDelaySeconds` to 120 to give Cassandra enough time to finish startup.
- Pinned the pod to `redpanda-lab-worker` (the node at ~48% memory allocation) away from `worker2` which was at ~94% due to Redpanda.

---

### Problem 2 — StatefulSet Not Rolling Out Config Changes

**Symptom:** Config changes pushed to git but the Cassandra pod kept running with the old spec.

**Root cause:** The Cassandra subchart defaults to `updateStrategy: OnDelete`. ArgoCD synced the new spec to the StatefulSet, but Kubernetes won't apply `OnDelete` updates until the pod is manually deleted.

**Fix:** Added `updateStrategy: type: RollingUpdate` to `values.yaml` so config changes automatically roll out.

---

### Problem 3 — Visibility Store Not Configured (Server CrashLoop)

**Symptom:** All Temporal server pods (`frontend`, `history`, `matching`, `worker`) crashed with:
```
config validation error: persistence config error: datastore "visibility":
must provide config for one and only one datastore: elasticsearch, cassandra, sql or custom store
```

**Root cause:** This is a design gap in the Temporal Helm chart (`temporal 0.48.0`). The chart's `server-configmap.yaml` template only has branches for Elasticsearch and SQL in the visibility store section:

```yaml
visibility:
{{- if or $elasticsearch.enabled $elasticsearch.external }}
  elasticsearch: ...
{{- else if eq (include "temporal.persistence.driver" ...) "sql" }}
  sql: ...
{{- end }}
```

There is **no Cassandra branch** for visibility. The chart defaults `visibility.driver: "cassandra"` in `values.yaml`, but nothing is rendered for it. This leaves the visibility section empty, and Temporal refuses to start.

**Fix:**
- Switched the visibility store to PostgreSQL (`postgres12` SQL driver), which IS supported by the chart template.
- Deployed a minimal PostgreSQL pod (`temporal-visibility-postgresql`) in the `temporal` namespace via a raw Kubernetes manifest (`temporal/manifests/postgres-visibility.yaml`).
- Added `temporal/manifests` as a third ArgoCD source so the PostgreSQL resources are deployed alongside the Helm chart.

---

### Problem 4 — ArgoCD SyncError on Immutable Job spec.template

**Symptom:** ArgoCD showed `SyncError` after the Cassandra image tag was changed from `3.11.3` to `4.0`:
```
Job.batch "temporal-schema-1" is invalid: spec.template: ... field is immutable
```

**Root cause:** Kubernetes Jobs have an immutable `spec.template`. When the Cassandra image tag changed, the schema Job's init container image also changed, and ArgoCD tried to patch the running Job — which Kubernetes rejects.

**Fix:** Added `ignoreDifferences` in `app.yaml` so ArgoCD never tries to diff or patch a Job's `spec.template` after creation:

```yaml
ignoreDifferences:
  - group: batch
    kind: Job
    jqPathExpressions:
      - .spec.template
```

When the schema Job needs to be updated (e.g. after a Cassandra image change), delete it manually and ArgoCD will recreate it fresh on the next sync.

---

### Problem 5 — app.yaml Changes Not Self-Applying

**Symptom:** Changes to `temporal/app.yaml` pushed to git had no effect on the live cluster.

**Root cause:** The `temporal/app.yaml` file *defines* the ArgoCD Application, but ArgoCD does not manage itself. There is no "app of apps" here, so `app.yaml` must be applied manually each time its structure changes (sources, ignoreDifferences, syncPolicy, etc.).

**Fix:** Apply manually after any structural change to `app.yaml`:
```bash
kubectl apply -f temporal/app.yaml
```

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
- ArgoCD installed (`argocd` namespace)
- This gitops repo accessible from the cluster
- At least **4 Gi free memory** on the target node (Cassandra 2 Gi + PostgreSQL 512 Mi + Temporal server pods)

---

### Step 1 — Pin Cassandra to the right node

Edit `temporal/values.yaml` and set the `nodeSelector` to a node that has enough free memory:

```yaml
cassandra:
  selector:
    nodeSelector:
      kubernetes.io/hostname: <your-worker-node-name>
```

Check which node has the most free memory:
```bash
kubectl describe nodes | grep -A5 "Allocated resources"
```

---

### Step 2 — Apply the ArgoCD Application

```bash
kubectl apply -f temporal/app.yaml
```

This creates the ArgoCD Application resource in the cluster. ArgoCD will then begin syncing automatically.

> **Important:** You must re-run this command any time you change the structure of `app.yaml` itself (sources, ignoreDifferences, syncPolicy). Regular value changes in `values.yaml` or manifests in `temporal/manifests/` are picked up automatically by ArgoCD without needing to re-apply `app.yaml`.

---

### Step 3 — Wait for Cassandra to become Ready

Cassandra takes ~2–3 minutes to start. Watch it:

```bash
kubectl get pods -n temporal -w
```

Wait until `temporal-cassandra-0` shows `1/1 Running` before continuing.

---

### Step 4 — Wait for the Schema Job to Complete

After Cassandra is Ready, ArgoCD will run the schema Job (`temporal-schema-1`). This Job:
1. Creates Cassandra keyspaces (`temporal`, `temporal_visibility` for PostgreSQL)
2. Runs schema migrations for both datastores

```bash
kubectl get job -n temporal
# Wait until temporal-schema-1 shows COMPLETIONS: 1/1
```

If the schema Job fails or gets stuck, delete it and trigger a sync — ArgoCD will recreate it:
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

Expected output:
```
temporal-admintools-xxx             1/1   Running    
temporal-cassandra-0                1/1   Running    
temporal-frontend-xxx               1/1   Running    
temporal-history-xxx                1/1   Running    
temporal-matching-xxx               1/1   Running    
temporal-schema-1-xxx               0/1   Completed  
temporal-visibility-postgresql-xxx  1/1   Running    
temporal-web-xxx                    1/1   Running    
temporal-worker-xxx                 1/1   Running    
```

---

### Step 6 — Verify Temporal is working

Use the admin tools pod:
```bash
kubectl exec -it -n temporal deployment/temporal-admintools -- tctl namespace list
```

Or check the Web UI (if you have an Ingress or port-forward):
```bash
kubectl port-forward -n temporal svc/temporal-web 8088:8088
# Open http://localhost:8088
```

---

## Troubleshooting

### Cassandra keeps crashing (OOMKilled)

Check which node it landed on and how much memory is free:
```bash
kubectl get pod temporal-cassandra-0 -n temporal -o wide
kubectl describe node <node-name> | grep -A5 "Allocated resources"
```

If the node is overcommitted, pin Cassandra to a less busy node via `nodeSelector` in `values.yaml`.

---

### Server pods crash with "must provide config for one and only one datastore"

This means the visibility store is not configured. Verify `values.yaml` has:
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

Also verify the PostgreSQL pod is running:
```bash
kubectl get pods -n temporal | grep postgresql
```

---

### ArgoCD shows SyncError for Job spec.template is immutable

The schema Job's spec changed (e.g. after a Cassandra image tag update). Delete the Job and sync:
```bash
kubectl delete job temporal-schema-1 -n temporal
kubectl -n argocd patch application temporal --type merge \
  -p '{"operation":{"sync":{"syncStrategy":{"hook":{"force":true}}}}}'
```

---

### Server pods start but then crash with "unable to read DB schema version"

The server started before the schema Job finished setting up the PostgreSQL visibility schema. The pods will recover on their next restart once the schema Job completes. Speed it up:
```bash
kubectl delete pod -n temporal \
  $(kubectl get pods -n temporal -l app.kubernetes.io/component=frontend,history,matching,worker \
    --no-headers -o name | tr '\n' ' ')
```
Or just delete the specific pods that are in CrashLoopBackOff.

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                  Temporal Namespace                  │
│                                                      │
│  ┌──────────────┐    ┌──────────────────────────┐   │
│  │  Cassandra   │    │   PostgreSQL (visibility) │   │
│  │  (default    │    │   temporal_visibility DB  │   │
│  │   store)     │    │   (workflow search/list)  │   │
│  └──────┬───────┘    └─────────────┬────────────┘   │
│         │                          │                 │
│  ┌──────▼──────────────────────────▼──────────────┐ │
│  │         Temporal Server (4 services)            │ │
│  │   frontend  │  history  │  matching  │  worker  │ │
│  └─────────────────────────────────────────────────┘ │
│                                                      │
│  ┌─────────────┐   ┌──────────────┐                 │
│  │  admin-tools │   │   web UI     │                 │
│  └─────────────┘   └──────────────┘                 │
└─────────────────────────────────────────────────────┘
```

- **Cassandra** stores workflow state (default store) — durable, append-heavy
- **PostgreSQL** stores visibility records (search indexes for workflow listing)
- The Helm chart (v0.48.0) does not support Cassandra as the visibility store; PostgreSQL is the lightweight alternative to Elasticsearch
