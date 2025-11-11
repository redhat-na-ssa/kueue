# Kueue Setup Guide for OpenShift
## 1. Prerequisites

A supported OpenShift Container Platform cluster (4.x) with cluster-admin permissions. 
Red Hat Docs
+1

oc CLI configured and connected to the cluster.

The cert-manager Operator installed (if required by the Kueue Operator). 
Red Hat Docs
+1

Ensure resource quotas, node pools etc are provisioned for batch jobs.

## 2. Install the Kueue Operator on OpenShift

In the OpenShift console, go to Operators → OperatorHub.

Find and select Red Hat build of Kueue Operator. 

Click Install, choose the desired namespace (e.g., openshift-kueue-operator), select installation mode (e.g., “All namespaces” or a specific project), and install. 

After installation, navigate to Operators → Installed Operators, and verify the status is Succeeded. 

## 2.2 Via YAML or CLI (if OperatorHub not available)

If you prefer or need to use raw manifests (less common on OpenShift), you could apply the upstream Kueue manifests:

```yaml
oc apply --server-side -f https://github.com/kubernetes-sigs/kueue/releases/download/vX.Y.Z/manifests.yaml
```


However — for OpenShift environments it’s better to use the Red Hat build/operator for full compatibility.

## 3. Create a Kueue Custom Resource (CR)

After the Operator is installed, you need to create a Kueue CR to configure the installation. Example YAML:

```yaml
apiVersion: kueue.openshift.io/v1
kind: Kueue
metadata:
  name: cluster
  namespace: openshift-kueue-operator
spec:
  managementState: Managed
  config:
    integrations:
      frameworks:
        - BatchJob
    preemption:
      preemptionPolicy: Classical
```

In this example:

name must be cluster (as required by the Red Hat build). 
Red Hat Docs

managementState: Managed means the Operator manages Kueue.

integrations.frameworks lists workload types Kueue should integrate with (BatchJob, MPI, etc).

preemptionPolicy set to Classical (or FairSharing if you want fair-sharing mode) 
Red Hat Docs

Apply with:
```yaml
oc apply -f kueue-cr.yaml
oc get kueue -n openshift-kueue-operator
```

## 4. Label Namespaces to Opt-in

The Operator uses a webhook to only manage jobs in namespaces marked for Kueue. You must add a label to namespaces you want Kueue to target:
```yaml
oc label namespace <namespace> kueue.openshift.io/managed=true
```


## 5. Configure ResourceFlavors, ClusterQueue, LocalQueue
## 5.1 ResourceFlavor

Represents a “flavor” of compute (node type, taints/labels). Example:
```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ResourceFlavor
metadata:
  name: default-flavor
```
### Optionally specify node labels/taints to associate this flavor


5.2 ClusterQueue

Defines quotas and scheduling policy cluster-wide:

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata:
  name: cluster-queue
spec:
  namespaceSelector: {}   # allow all namespaces
  preemption:
    withinClusterQueue: LowerPriority
  resourceGroups:
    - coveredResources: ["cpu", "memory"]
      flavors:
        - name: default-flavor
          resources:
            - name: "cpu"
              nominalQuota: 1000
            - name: "memory"
              nominalQuota: 256Gi

```

## 5.3 LocalQueue

Defines a per-namespace queue that maps to the above ClusterQueue:
```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: LocalQueue
metadata:
  namespace: team-namespace
  name: user-queue
spec:
  clusterQueue: cluster-queue
```


## 6. Submitting a Job

Once config is in place and namespace labelled, you can submit a Job (or another supported workload) to be managed by Kueue. Example:
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: sample-job-
  namespace: team-namespace
  annotations:
    kueue.x-k8s.io/queue-name: user-queue
spec:
  parallelism: 1
  completions: 1
  template:
    spec:
      containers:
      - name: example
        image: registry.example/batch/calc:latest
        args: ["30s"]
        resources:
          requests:
            cpu: 1
      restartPolicy: Never
```

Kueue will intercept via webhook (in the labelled namespace) and handle admission based on your queue/cluster queue/quota settings.

## 7. Monitoring & Metrics

The Operator and Kueue components expose metrics suitable for Prometheus scraping.

On OpenShift, you can monitor via oc get pods -n <namespace> and check CRDs like clusterqueues.kueue.x-k8s.io.

Ensure you have RBAC permissions (role kueue-batch-admin-role) if managing Kueue objects. 

## 8. Uninstallation (if needed)

Delete the Kueue CR first (in the Operator namespace).

Uninstall the Operator via Operators → Installed Operators → Uninstall. On uninstall you may choose to remove operand instances or keep them. 
Red Hat Docs

Remove the label kueue.openshift.io/managed=true from any namespaces if you’re removing Kueue.

Remove any ClusterQueues / LocalQueues / ResourceFlavors you created.

## 9. Full Sample Markdown

Here is a full snippet you can copy into a Markdown file (e.g., kueue-openshift-setup.md):

# Install Kueue on OpenShift

## 1. Install Operator  
Use the OperatorHub in the OpenShift Console:
1. Navigate to Operators → OperatorHub  
2. Select **Red Hat build of Kueue Operator**  
3. Install to namespace `openshift-kueue-operator`, for all namespaces or restricted as desired  

## 2. Create Kueue CR  
```yaml
apiVersion: kueue.openshift.io/v1
kind: Kueue
metadata:
  name: cluster
  namespace: openshift-kueue-operator
spec:
  managementState: Managed
  config:
    integrations:
      frameworks:
        - BatchJob
    preemption:
      preemptionPolicy: Classical
```

## 3. Label Namespaces
oc label namespace team-namespace kueue.openshift.io/managed=true

## 4. Configure ResourceFlavor / ClusterQueue / LocalQueue
```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ResourceFlavor
metadata:
  name: default-flavor
---
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata:
  name: cluster-queue
spec:
  namespaceSelector: {}
  preemption:
    withinClusterQueue: LowerPriority
  resourceGroups:
    - name: default
      flavors:
        - name: default-flavor
          quota:
            min: 1000
---
apiVersion: kueue.x-k8s.io/v1beta1
kind: LocalQueue
metadata:
  namespace: team-namespace
  name: user-queue
spec:
  clusterQueue: cluster-queue
```
## 5. Submit a Job
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: sample-job-
  namespace: team-namespace
  annotations:
    kueue.x-k8s.io/queue-name: user-queue
spec:
  parallelism: 3
  completions: 3
  template:
    spec:
      containers:
      - name: example
        image: registry.example/batch/calc:latest
        args: ["30s"]
        resources:
          requests:
            cpu: 1
      restartPolicy: Never
```
## 6. Monitoring & Uninstall

Check pods: oc get pods -n openshift-kueue-operator

To uninstall: Remove Kueue CR → Uninstall Operator via console → Clean up resources & labels

------------------
# Preemption 
## 1) Create a project
oc new-project kueue-demo

## 2) PriorityClasses (high vs low)
priorityclasses.yaml
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000
preemptionPolicy: PreemptLowerPriority
globalDefault: false
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
preemptionPolicy: PreemptLowerPriority
globalDefault: false
```
```
oc apply -f priorityclasses.yaml
```
## 3) ResourceFlavor (default)
resourceflavor.yaml
```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ResourceFlavor
metadata:
  name: default-flavor
spec: {}
```
```
oc apply -f resourceflavor.yaml
```
## 4) One ClusterQueue (with preemption enabled)

Small quota (CPU=2) so the low-priority job fully occupies it.

withinClusterQueue: LowerPriority allows a higher-priority workload to preempt a lower-priority one in the same CQ. 
Kueue

clusterqueue.yaml
```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata:
  name: cq-preempt
spec:
  namespaceSelector: {}
  queueingStrategy: StrictFIFO
  flavorFungibility:
    whenCanBorrow: Borrow          # ok
    whenCanPreempt: Preempt        # ✅ use Preempt or TryNextFlavor
  preemption:
    withinClusterQueue: LowerPriority
  resourceGroups:
    - coveredResources: ["cpu","memory"]
      flavors:
        - name: default-flavor
          resources:
            - name: "cpu"
              nominalQuota: 2
            - name: "memory"
              nominalQuota: 4Gi

```
```
oc apply -f clusterqueue.yaml
```

Fields withinClusterQueue, reclaimWithinCohort, and borrowWithinCohort.policy are the current v1beta1 knobs for preemption behavior. 
Kueue

## 5) Two LocalQueues (A and B) in the same namespace

Both point to the same ClusterQueue; workloads are admitted according to CQ policy. 
Kueue

localqueues.yaml
```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: LocalQueue
metadata:
  name: queue-a
  namespace: kueue-demo
spec:
  clusterQueue: cq-preempt
---
apiVersion: kueue.x-k8s.io/v1beta1
kind: LocalQueue
metadata:
  name: queue-b
  namespace: kueue-demo
spec:
  clusterQueue: cq-preempt
```
```
oc apply -f localqueues.yaml
```
## 6) Low-priority job in Queue A (fills all quota)
job-low.yaml
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: low-job
  namespace: kueue-demo
  labels:
    kueue.x-k8s.io/queue-name: queue-a   # <-- submit to queue A
spec:
  template:
    spec:
      restartPolicy: Never
      priorityClassName: low-priority
      containers:
      - name: main
        image: busybox
        command: ["sh","-c","echo LOW running; sleep 300"]
        resources:
          requests:
            cpu: "2"        # consumes the full CQ quota
            memory: "512Mi"
          limits:
            cpu: "2"
            memory: "512Mi"
```
```
oc apply -f job-low.yaml
oc get pods -n kueue-demo -w   # wait until low-job pod is Running
```
## 7) High-priority job in Queue B (preempts Queue A’s job)
job-high.yaml
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: high-job
  namespace: kueue-demo
  labels:
    kueue.x-k8s.io/queue-name: queue-b   # <-- submit to queue B
spec:
  template:
    spec:
      restartPolicy: Never
      priorityClassName: high-priority
      containers:
      - name: main
        image: busybox
        command: ["sh","-c","echo HIGH running; sleep 30; echo done"]
        resources:
          requests:
            cpu: "2"        # needs the same 2 CPUs
            memory: "512Mi"
          limits:
            cpu: "2"
            memory: "512Mi"
```
```
oc apply -f job-high.yaml
```
## 8) Watch the preemption happen
Kueue’s view (workloads)
```
oc get workloads -A
oc describe clusterqueue cq-preempt
```
Pod state changes (low-job should be terminated; high-job should run)
```
oc get pods -n kueue-demo -w
```

You’ll see high-job admitted and running; low-job gets preempted because of higher priority under withinClusterQueue: LowerPriority. 
Kueue

