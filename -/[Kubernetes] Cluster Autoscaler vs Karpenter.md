# 💚 Cluster Autoscaler vs Karpenter

## 💛 In Short
- Both are node autoscalers: they add nodes when pods are stuck in `Pending`
- **CA (Cluster Autoscaler):** resizes node groups you created in advance
- **Karpenter:** launches EC2 instances directly to fit what the pods need
- Both decide from pod **requests**, not actual usage. If requests are wrong, both make bad decisions
- HPA scales the number of pods. These two scale the number of nodes
## 💛 Why Do We Need It?
- When pods increase and no node has room, they stay in `Pending`
- Keeping spare nodes around all the time wastes money, so capacity has to follow demand
## 💛 How Does It Work?
### 🤍 CA Request Flow
```
Pod is Pending
→ CA simulates scheduling against node group templates
→ An expander picks which group to grow
→ ASG desired capacity goes up
→ ASG launches EC2 → node joins → pod is scheduled
```
- CA assumes all nodes in a group have the same shape. If you need a different shape, add another group
- Scale-down: CA removes a node when its requested utilization stays under 50% (default) for 10 minutes and its pods can move elsewhere
### 🤍 Karpenter Request Flow
```
Pod is Pending
→ Karpenter batches Pending pods and reads their requests, affinity, and tolerations
→ It works out which instance types fit within the NodePool's limits
→ It creates a NodeClaim and launches the instance through the EC2 API
→ Node joins → pod is scheduled
```
- **Consolidation:** removes nodes or replaces them with cheaper ones
  - `WhenEmpty`: only removes empty nodes. The most conservative option
  - `Balanced`: acts only when the savings outweigh the pod disruption. This comes from the latest docs, so check that your version supports it
  - `WhenEmptyOrUnderutilized`: acts whenever it can lower cost. The default
- **Drift:** replaces nodes when the NodePool, EC2NodeClass, or AMI changes. Useful during cluster upgrades
- **Expiration:** replaces nodes after `expireAfter`
- **Disruption budget:** limits how many nodes can be replaced at once. The default is `nodes: 10%`, and `schedule` uses UTC
- **Spot:** receives interruption notices through an SQS queue and drains ahead of time
## 💛 Comparison
## 💛 When to Use Which
- **Karpenter:** you want mixed instance types, heavy Spot use, tight bin-packing, or fewer node groups to manage
- **CA:** you need fixed node shapes, you already run node groups, or your cloud has no Karpenter provider
- **Main trade-off:** Karpenter saves money by moving pods more often, so PDBs, graceful shutdown, and accurate requests are required. CA is slower and packs less tightly, but it changes less on its own
## 💛 Example
### 🤍 Karpenter NodePool
```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: general
spec:
  template:
    spec:
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]   # Wide range gives Spot more replacement options
      expireAfter: 720h
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
    budgets:
      - nodes: "10%"                 # Normal limit on concurrent replacements
      - nodes: "0"                   # No replacements for 8 hours from 09:00 UTC on weekdays
        schedule: "0 9 * * mon-fri"
        duration: 8h
  limits:
    cpu: "1000"                      # Total CPU this pool can create
```
### 🤍 CA Key Flags
```bash
--node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/my-cluster
--expander=least-waste                  # Pick the group that wastes the least
--balance-similar-node-groups           # Keep similar groups balanced across AZs
--scale-down-utilization-threshold=0.5
--scale-down-unneeded-time=10m
```
## 💛 Operational Tips
- Run the Karpenter controller somewhere it doesn't manage (a managed node group or Fargate), so it can't remove its own node
- Versions: match CA to the cluster's minor version. Karpenter has its own compatibility matrix
- Helm doesn't upgrade CRDs on a chart upgrade. For Karpenter, upgrade the CRDs first with the `karpenter-crd` chart
- Add the `karpenter.sh/do-not-disrupt: "true"` annotation to pods that must not be interrupted, which excludes them from voluntary disruption
- PDBs only pace voluntary disruptions such as consolidation. For Spot reclaims, rely on topology spread and a wider range of instance types
## 💛 Follow-up Questions
- "Pods are Pending but no nodes are being added"
  - CA: no group template fits, or the ASG hit its max
  - Karpenter: NodePool requirements or `limits`, too few subnet IPs, or no capacity
  - Check: `kubectl describe pod`, NodeClaim status, controller logs
- "Nodes aren't scaling down"
  - A PDB with ALLOWED DISRUPTIONS at 0, a do-not-disrupt annotation, or a budget window set to 0
  - For CA, also check for `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"` and pods that can't move
- "Is Karpenter always cheaper?"
  - No. If requests are inflated, the waste stays, and moving pods more often has its own cost
- "Can you run both?"
  - They can compete over the same Pending pods. Most teams run both only during a migration, then remove CA
## 💛 30-Second Answer
> CA resizes the ASGs behind node groups you defined in advance. Karpenter looks at what Pending pods need and launches EC2 instances directly. That gives Karpenter a wider instance choice and lower cost through consolidation, but it moves pods more often, so PDBs and graceful shutdown are required.
## 💛 References
- Karpenter NodePools
- Karpenter Disruption
- Migrating from Cluster Autoscaler
- Cluster Autoscaler FAQ
- EKS Best Practices: Karpenter
- EKS Best Practices: Cluster Autoscaler
