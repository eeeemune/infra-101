# 💚 Kubernetes Fundamentals

## 💛 Workloads: when to use what
- **Deployment:** apps whose pods are interchangeable. APIs, web servers. A rolling update works by creating a new ReplicaSet.
- **StatefulSet:** when each pod needs its own identity. Databases, Kafka.
  - Stable names (`db-0`, `db-1`).
  - Each pod keeps its own PVC. After a restart it reattaches to the same volume.
  - By default pods start in order and terminate in reverse order.
  - Pair it with a Headless Service when you need per-pod DNS.
- **DaemonSet:** one pod per node. Log collection, CNI, node-exporter.
- **Job:** a task that finishes. Completes on success, retries up to `backoffLimit` on failure. A CronJob creates Jobs on a schedule.
### 🤍 Follow-up questions
- "Why not run a database as a Deployment?" The pod names and volumes are not stable, so it does not fit systems where members must be told apart, like a replicated cluster.
- "If I scale a StatefulSet down, what happens to the PVCs?" By default they stay (`persistentVolumeClaimRetentionPolicy` defaults to Retain).
- "What if I run a batch job as a Deployment?" `restartPolicy: Always` restarts it even after a clean exit (exit 0), so it falls into CrashLoopBackOff.
## 💛 Service types: where traffic comes in
- **ClusterIP (default):** a virtual IP reachable only inside the cluster.
- **NodePort:** opens the same port on every node (default 30000 to 32767). Sits on top of ClusterIP.
- **LoadBalancer:** provisions a cloud load balancer and wires it up. Usually sits on top of NodePort.
- **ExternalName:** returns a DNS CNAME only, no proxying.
- **Headless** (`clusterIP: None`): not a type but a variant of ClusterIP. No virtual IP; DNS returns the pod IPs directly.
### 🤍 Request Flow
```
client → LB → node port → kube-proxy rules (or eBPF) → pod IP
                              ↑
              only Ready pods in the EndpointSlice are targets
```
### 🤍 Follow-up questions
- "Is Ingress a Service type?" No. It is an L7 routing rule handled by a controller.
- "How do I preserve the client IP?" `externalTrafficPolicy: Local`. The catch: a node with no matching pod will not receive traffic.
## 💛 Probes: what happens when one fails
- **liveness:** on failure the kubelet restarts the container. "It is dead, bring it back."
- **readiness:** on failure the pod is pulled from the Service targets. No restart. "Do not send traffic right now."
- **startup:** blocks the other two until it passes. Past its failure count the container restarts. For slow-booting apps.
### 🤍 Example
```yaml
startupProbe:
  httpGet: { path: /healthz, port: 8080 }
  periodSeconds: 5
  failureThreshold: 60   # wait up to 5 minutes for boot
livenessProbe:
  httpGet: { path: /healthz, port: 8080 }   # check only the process itself
  periodSeconds: 10
readinessProbe:
  httpGet: { path: /ready, port: 8080 }     # may also check dependencies
  periodSeconds: 5
```
### 🤍 Follow-up questions
- "What if I put a DB check in liveness?" A brief DB slowdown restarts the whole pod and makes the outage worse. Check dependencies in readiness instead.
- "What if liveness and readiness are identical?" The pod can be restarted without the chance to drop out of rotation first. Split the roles.
## 💛 cordon, drain, taint: how you move pods off a node
- **cordon:** blocks new pods only. Existing pods stay.
- **drain:** cordon plus evicting existing pods through the Eviction API. Respects PDBs.
- **taint:** the node marks itself "do not come here without a matching toleration."
  - `NoSchedule`: no new pods.
  - `PreferNoSchedule`: avoid if possible.
  - `NoExecute`: evicts existing pods too.
### 🤍 Example
```bash
kubectl cordon node-1
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data
kubectl taint nodes node-1 gpu=true:NoSchedule
```
### 🤍 Follow-up questions
- "Does adding a toleration send the pod to that node?" No. A toleration only permits, it does not attract. To place it there, add a nodeSelector or affinity.
- "My drain is stuck." Usually a PDB. Check whether ALLOWED DISRUPTIONS is 0 in `kubectl get pdb -A`.
- "Does NoExecute respect PDBs?" No. It does not go through the Eviction API.
## 💛 CrashLoopBackOff: a state, not an error
- The container keeps exiting and the kubelet keeps stretching the restart delay. It starts at 10 seconds and doubles up to 5 minutes.
- The name does not tell you the cause, so read the exit code and the previous logs.
### 🤍 Example
```bash
kubectl describe pod <pod>      # Last State, Exit Code, Reason, Events
kubectl logs <pod> --previous   # logs from the container just before it died
```
### 🤍 Narrowing by exit code
- **1:** app error. Often a missing env var or config.
- **126 / 127:** cannot execute the command / command not found. A typo in command or entrypoint.
- **137:** SIGKILL. If `Reason: OOMKilled`, it ran out of memory. Do not assume OOM from 137 alone.
- **143:** SIGTERM. Common when the kubelet terminates the container after a liveness failure (confirm in Events).
- **0:** a clean exit that restarts anyway because of `restartPolicy: Always`. A finish-and-exit program run as a Deployment.
### 🤍 Follow-up questions
- "How is this different from ImagePullBackOff?" There the image could not be pulled, so the container never even started.
- "What about exec format error?" The image architecture does not match the node (for example an arm64 image on an amd64 node).
## 💛 Secrets: base64 is not encryption
- The base64 in the `data` field is just encoding. Anyone can restore it with `base64 -d`.
- Encryption is a separate thing you enable for data at rest in etcd.
### 🤍 Envelope encryption flow
```
Secret → encrypted with a DEK (data key) → stored in etcd
DEK → encrypted with the KEK (master key) in KMS → stored alongside
```
- The DEK is wrapped once more by the KMS key, which is why it is called an "envelope." The key lives outside etcd, so leaking an etcd backup alone does not expose plaintext.
- **EKS:** on Kubernetes 1.28 and later all API data gets envelope encryption by default. The default is an AWS-owned key; you can switch to a customer managed key (CMK).
### 🤍 Follow-up questions
- "So are Secrets safe now?" Only at rest. Anyone with `get secrets` permission reads plaintext through the API. RBAC scope is what matters.
- "What are the other leak paths?" Env vars leaking into logs or crash dumps, and manifests committed to Git. Keep secrets out of Git with an external store (External Secrets and similar).
## 💛 References
- Workloads
- Service
- Liveness, Readiness, Startup Probes
- Safely Drain a Node
- Taints and Tolerations
- Pod Lifecycle: Container restarts
- Secrets
- Encrypting Secret Data at Rest
- EKS default envelope encryption
