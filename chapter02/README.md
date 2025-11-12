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