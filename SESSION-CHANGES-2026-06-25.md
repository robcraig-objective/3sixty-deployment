# 3Sixty Deployment — Session Change Log

**Date:** 2026-06-25
**Operator:** Robert Craig
**Cluster:** Docker Desktop (single node `docker-desktop`, Kubernetes v1.34.1)
**Namespace:** `threesixty`
**Helm release:** `threesixty-stack` (chart `threesixty-stack-1.5.0`, app version 5.2.5)

---

## 1. Objective

Roll all pods to pick up the latest deployment and bring the cluster to a fully healthy state.
On inspection, the roll surfaced two pre-existing latent issues that were fixed during the session.

---

## 2. Problems Found

| # | Component | Symptom | Root Cause |
|---|-----------|---------|-----------|
| 1 | `hybridsearch-remoteagent` | `CrashLoopBackOff`, 1674 restarts, `OOMKilled` (exit 137) | Memory limit of **256Mi** too small for a Java 17 / Spring Boot app. JVM could not start within the cgroup limit. |
| 2 | `mongo` & `opensearch` | Roll deadlocked — old + new pods both `Running`/`Pending`, duplicate ReplicaSets | Default `RollingUpdate` strategy on single-replica **ReadWriteOnce** `hostpath` volumes. New pod cannot acquire the data lock (mongo WiredTiger) / cannot fit in node memory (opensearch) until the old pod exits, but RollingUpdate won't remove the old pod until the new one is Ready. |

---

## 3. Changes Made

### 3.1 File changes (committed to chart sources — require review/commit)

Three files changed, 10 insertions(+), 2 deletions(-):

#### `helm/charts/hybridsearch/values.yaml`
Increased remoteagent memory to give the JVM headroom.

```diff
   resources:
     requests:
-      memory: "128Mi"
+      memory: "512Mi"
       cpu: "100m"
     limits:
-      memory: "256Mi"
+      memory: "1Gi"
       cpu: "500m"
```

#### `helm/charts/mongo/templates/deployment-mongo.yaml`
Added `Recreate` strategy (single-replica RWO database).

```diff
 spec:
   replicas: {{ .Values.replicaCount }}
+  # Recreate: single-replica DB on a ReadWriteOnce volume. RollingUpdate would
+  # start a second pod that cannot acquire the WiredTiger data lock, deadlocking the roll.
+  strategy:
+    type: Recreate
   selector:
```

#### `helm/charts/opensearch/templates/deployment-opensearch.yaml`
Added `Recreate` strategy (single-replica RWO search node).

```diff
 spec:
   replicas: 1
+  # Recreate: single-replica search node on a ReadWriteOnce volume. RollingUpdate would
+  # schedule a second pod the node lacks memory for and that cannot share the data dir.
+  strategy:
+    type: Recreate
   selector:
```

### 3.2 Live cluster actions (already applied to the running cluster)

| Action | Command (summary) | Purpose |
|--------|-------------------|---------|
| Helm upgrade | `helm upgrade threesixty-stack . -n threesixty --reuse-values --set hybridsearch.remoteagent.resources...=1Gi/512Mi` | Applied new remoteagent memory limit live (Helm revision 4). |
| Roll all deployments | `kubectl -n threesixty rollout restart deployment` | Restarted all 12 deployments to pick up latest. |
| Delete stale old pods | `kubectl -n threesixty delete pod <old-opensearch> <old-mongo>` | Freed node memory and released mongo data lock to break the initial roll stall. |
| Patch strategy live | `kubectl -n threesixty patch deploy mongo/opensearch --type merge -p '{"spec":{"strategy":{"type":"Recreate"}}}'` | Broke the RollingUpdate deadlock immediately (mirrors the template change above). |

> **Note:** The `--set` flags on the helm upgrade match the values.yaml edit. A future
> `helm upgrade` from the edited chart will produce the same result without `--set`.

---

## 4. Verification

### 4.1 Final pod state — all 12 Running & Ready

```
admin                       Ready
discovery                   Ready
hybridsearch-3sixtyrag      Ready
hybridsearch-oirag          Ready
hybridsearch-ollama         Ready
hybridsearch-remoteagent    Ready   <- was CrashLoopBackOff, now 0 restarts after fix
mongo                       Ready
opensearch                  Ready
opensearch-dashboards       Ready
rabbitmq                    Ready
scim-server                 Ready
traefik                     Ready
```

(Elevated restart counts on mongo/admin are residual from the deadlock recovery, not ongoing failures — both stable post-fix.)

### 4.2 remoteagent memory limit applied
```
limits:   { cpu: 500m, memory: 1Gi }
requests: { cpu: 100m, memory: 512Mi }
```

### 4.3 3sixty-admin `/data` volume CRUD permission check

PVC `threesixty-stack-admin-data` mounted read-write at `/data`; admin runs as uid=0; `/data` is mode 0777.
In-container test results:

| Operation | Result |
|-----------|--------|
| Create | PASS |
| Read | PASS |
| Update (append) | PASS |
| Delete | PASS |
| Subdirectory create/delete | PASS |

**Conclusion:** the application has full create/read/update/delete access to its `/data` volume. No permission issue.

---

## 4b. Traefik ingress — persistent localhost access (added later in session)

**Problem:** The bundled Traefik `LoadBalancer` (ports 7070/7443) bound the host ports on
Docker Desktop but did not route traffic into the cluster — the ingress was unreachable
(`ERR_CONNECTION_CLOSED`). The only thing serving those ports was a transient `kubectl port-forward`.

**Fix (durable):** Changed the Traefik service to **NodePort** with pinned ports, in
`helm/values-production.yaml` (the active values file; note: gitignored, holds credentials):

```diff
 traefik:
   ports:
     web:
       exposedPort: 7070
+      nodePort: 30070
-      http:
-        redirections:
-          entryPoint:
-            to: websecure
-            scheme: https
     websecure:
       exposedPort: 7443
+      nodePort: 30443
   service:
-    type: LoadBalancer
+    type: NodePort
```

Applied via `helm upgrade threesixty-stack . -n threesixty -f values-production.yaml` (revision 5).

**Result — admin reachable on localhost, survives pod/cluster restarts/reboots:**
- http://localhost:30070/3sixty-admin  (plain HTTP, no cert warning — `/login` returns 200)
- https://localhost:30443/3sixty-admin (HTTPS, self-signed cert warning)

> The HTTP→HTTPS redirect was removed for local convenience. **Re-enable it (and use a
> LoadBalancer/proper ingress) for any real/production deployment.**

---

## 5. Outstanding / Follow-up

- [ ] **Commit the three chart file changes** (currently working-tree edits only). The live cluster already reflects them via `--set`/patch, but the chart source must be committed so the changes survive the next clean `helm upgrade`.
- [ ] Consider bumping the chart version (`Chart.yaml`) when committing these template changes.
- [ ] (Optional) Review admin liveness/readiness probe `initialDelaySeconds` — admin WAR takes ~45s to deploy; under node load it tripped the probe once before settling.
