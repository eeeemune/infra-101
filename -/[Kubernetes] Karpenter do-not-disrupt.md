# 💚 Karpenter do-not-disrupt

## 💛 What is it?
**Karpenter** is a Kubernetes node autoscaler (common on EKS) that provisions and removes nodes to fit your pods. Left alone, it constantly reshapes the fleet: consolidating underused nodes, replacing drifted ones, and expiring old ones.
The `karpenter.sh/do-not-disrupt: "true"` annotation tells Karpenter "do not **voluntarily** disrupt this." You put it on a **pod** (which protects any node running it) or on a **node / NodeClaim** (which protects that node).
## 💛 Why do we need it?
Karpenter draining and deleting a node under your pods is usually fine, but sometimes it is not:
- A **long batch job** that would lose hours of work if its node vanished.
- A **stateful pod mid-operation** (a migration, a data load) that must not be moved.
- A **CI runner mid-build** or anything you do not want interrupted.
For those, `do-not-disrupt` **stops Karpenter from terminating the node while that pod runs.**
## 💛 What counts as disruption
This matters a lot. The annotation blocks only **voluntary** (Karpenter-initiated) disruption:
- **Consolidation**: bin-packing pods onto fewer or cheaper nodes.
- **Drift**: the node no longer matches its NodePool or NodeClass (for example a new AMI).
- **Expiration**: the node is older than its configured max lifetime.
It does **not** block involuntary or external termination:
- **Spot interruptions** and hardware failure.
- `kubectl delete node` or a manual drain.
- A node going **NotReady**.
So `do-not-disrupt` is not a promise the pod is never interrupted. It only stops Karpenter's own optimization from moving it.
## 💛 How to use it
### 🤍 On a pod (most common)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: long-job
  annotations:
    karpenter.sh/do-not-disrupt: "true"
```
Any node running at least one pod with this annotation is protected from voluntary disruption.
On a Deployment or Job, it goes in the **pod template**, not the top-level metadata:
```yaml
spec:
  template:
    metadata:
      annotations:
        karpenter.sh/do-not-disrupt: "true"
```
### 🤍 On a node or NodeClaim
```bash
kubectl annotate node ip-**** karpenter.sh/do-not-disrupt=true
```
This protects the whole node regardless of which pods it runs.
## 💛 The safety valve: terminationGracePeriod
In Karpenter v1, the NodePool field `spec.template.spec.terminationGracePeriod` sets a **hard cap**. When a node genuinely needs to go (drift, expiration, or a delete), Karpenter forcibly terminates it after this period **even if a do-not-disrupt pod is still running**. This stops one annotated pod from blocking an upgrade forever.
```yaml
spec:
  template:
    spec:
      terminationGracePeriod: 24h
```
Pair `do-not-disrupt` with a sensible `terminationGracePeriod` so protection has a ceiling.
## 💛 Note on older annotations
Before Karpenter v0.32, this was two separate annotations: `karpenter.sh/do-not-evict` (pods) and `karpenter.sh/do-not-consolidate` (nodes). Karpenter v1 merged them into the single `karpenter.sh/do-not-disrupt`. If you see the old ones, they are deprecated.
## 💛 Gotcha
- **It blocks ****only voluntary disruption****.** Spot reclaim, node failure, and `kubectl delete node` still take the node. Do not treat it as "this pod can never be interrupted."
- **It can block cost savings.** An underused node kept alive by one annotated pod cannot be consolidated, so you pay for a large node running one small protected pod. Use it narrowly and remove it when the work is done.
- **It can block upgrades and drift.** A node pinned by `do-not-disrupt` will not pick up AMI or config changes. Always pair it with `terminationGracePeriod` so it cannot stall forever.
- **Put it on the right object.** For a Deployment or Job it belongs in `spec.template.metadata.annotations` (the pod template), not the workload's own metadata.
- **Terminal pods stop protecting.** Once a Job pod completes, it no longer holds the node, so Karpenter is free to reclaim it.
## 💛 References
- Karpenter: Disruption concepts: https://karpenter.sh/docs/concepts/disruption/
- Karpenter: pod-level disruption controls (do-not-disrupt): https://karpenter.sh/docs/concepts/disruption/#pod-level-controls
- Karpenter: NodePools (terminationGracePeriod): https://karpenter.sh/docs/concepts/nodepools/
