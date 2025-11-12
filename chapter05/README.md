# Borrowing Resources
## 1) Create projects (namespaces)
```
oc new-project team-a
oc new-project team-b
```
## 2) Minimal ResourceFlavor
01_resourceflavor.yaml
```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ResourceFlavor
metadata:
  name: default-flavor
spec: {}
```
```
oc apply -f 01_resourceflavor.yaml
```
## 3) Two ClusterQueues in the same cohort with borrowing

Each team “owns” 2 CPUs (nominalQuota).

Each may borrow up to 2 CPUs (borrowingLimit) from the other when idle.

Both manage cpu and memory on the same flavor.

02_clusterqueues.yaml
```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata:
  name: cq-team-a
spec:
  cohort: cpu-shared
  namespaceSelector: {}                  # allow any namespace (we'll restrict via LQ)
  resourceGroups:
    - coveredResources: ["cpu","memory"]
      flavors:
        - name: default-flavor
          resources:
            - name: "cpu"
              nominalQuota: 2
              borrowingLimit: 2
            - name: "memory"
              nominalQuota: 4Gi
---
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata:
  name: cq-team-b
spec:
  cohort: cpu-shared
  namespaceSelector: {}
  resourceGroups:
    - coveredResources: ["cpu","memory"]
      flavors:
        - name: default-flavor
          resources:
            - name: "cpu"
              nominalQuota: 2
              borrowingLimit: 2
            - name: "memory"
              nominalQuota: 4Gi
```
```
oc apply -f 02_clusterqueues.yaml
```

Notes

cohort: cpu-shared is the key that allows sharing (borrowing) between these CQs.

borrowingLimit caps how much extra a CQ can use beyond its own nominalQuota when neighbors are idle.

## 4) LocalQueues for each namespace
03_localqueues.yaml
```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: LocalQueue
metadata:
  name: user-queue
  namespace: team-a
spec:
  clusterQueue: cq-team-a
---
apiVersion: kueue.x-k8s.io/v1beta1
kind: LocalQueue
metadata:
  name: user-queue
  namespace: team-b
spec:
  clusterQueue: cq-team-b
```
```
oc apply -f 03_localqueues.yaml
```
## 5) Team A submits a job that borrows

This job asks for 3 CPUs total (3 pods × 1 CPU) → it will consume its 2 owned CPUs and borrow 1 (if Team B is idle).

04_team-a-job.yaml
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: team-a-borrow-demo
  namespace: team-a
  labels:
    kueue.x-k8s.io/queue-name: user-queue
spec:
  parallelism: 3
  completions: 3
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: main
          image: busybox
          command: ["sh","-c","echo Team A running; sleep 120; echo done"]
          resources:
            requests:
              cpu: "1"
              memory: "256Mi"
            limits:
              cpu: "1"
              memory: "256Mi"
```
```
oc apply -f 04_team-a-job.yaml
```

Watch it admit and run (two pods from owned quota + one pod via borrow, assuming Team B is idle):
```
oc get workloads -A
oc get pods -n team-a -w
oc describe clusterqueue cq-team-a
oc describe clusterqueue cq-team-b
```

You should see cq-team-a showing usage beyond its nominal (i.e., borrowed), while cq-team-b remains idle/available.

## 6) Team B submits work (sharing in action)

Now submit 2 CPUs from Team B. If Team A is using borrowed capacity, Kueue will stop admitting more borrowing from A; once capacity is needed by B, new A workloads won’t be admitted beyond its nominal quota. (If you also want reclaim/preemption behavior to take borrowed capacity back, run Team B’s job with higher priority and enable preemption policy—see Notes at the end.)

team-b-job.yaml
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: team-b-nominal
  namespace: team-b
  labels:
    kueue.x-k8s.io/queue-name: user-queue
spec:
  parallelism: 2
  completions: 2
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: main
          image: busybox
          command: ["sh","-c","echo Team B running; sleep 60; echo done"]
          resources:
            requests:
              cpu: "1"
              memory: "256Mi"
            limits:
              cpu: "1"
              memory: "256Mi"
```
```
oc apply -f team-b-job.yaml
```

Observe sharing behavior:
```
oc get pods -n team-b -w
oc describe clusterqueue cq-team-a
oc describe clusterqueue cq-team-b
oc get workloads -A
```
What you should see

Team A’s first two pods run on its nominal quota; the 3rd pod is admitted via borrowing (while Team B is idle).

When Team B submits its job, it uses its own nominal quota. If total cluster capacity is tight, further admissions from A will not exceed its nominal; borrowing depends on B’s unused capacity.

Notes & variants

Preemption while sharing (optional):
If you also want Team B to reclaim capacity when A is using borrowed resources, add PriorityClasses (give Team B higher priority) and set the preemption policy in the CQs so borrowed work can be preempted. (The exact preemption knobs vary by Kueue version; if you tell me your Kueue version, I’ll give you a preemption-enabled variant that’s schema-correct for your build.)

GPU sharing:
Change coveredResources to ["nvidia.com/gpu"] and set nominalQuota / borrowingLimit on that resource. Your Job containers must request resources.requests.nvidia.com/gpu: 1.

Safer defaults:
Start with small borrowing limits (like 1) and increase after you see the behavior you want.