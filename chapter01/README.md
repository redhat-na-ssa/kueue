## 1. Create a Kueue Custom Resource (CR)

After the Operator is installed, you need to create a Kueue CR to configure the installation. Example YAML:

01_kueue-cr.yaml
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
oc apply -f 01_kueue-cr.yaml
oc get kueue -n openshift-kueue-operator
```

## 2. Label Namespaces to Opt-in

The Operator uses a webhook to only manage jobs in namespaces marked for Kueue. You must add a label to namespaces you want Kueue to target:
```yaml
oc label namespace <namespace> kueue.openshift.io/managed=true
```


## 3. Configure ResourceFlavors, ClusterQueue, LocalQueue
## 3.1 ResourceFlavor

Represents a “flavor” of compute (node type, taints/labels). Example:
```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ResourceFlavor
metadata:
  name: default-flavor
```
### Optionally specify node labels/taints to associate this flavor


3.2 ClusterQueue

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

## 3.3 LocalQueue

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


## 4. Submitting a Job

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

## 5. Monitoring & Metrics

The Operator and Kueue components expose metrics suitable for Prometheus scraping.

On OpenShift, you can monitor via oc get pods -n <namespace> and check CRDs like clusterqueues.kueue.x-k8s.io.

Ensure you have RBAC permissions (role kueue-batch-admin-role) if managing Kueue objects. 

## 6. Uninstallation (if needed)

Delete the Kueue CR first (in the Operator namespace).

Uninstall the Operator via Operators → Installed Operators → Uninstall. On uninstall you may choose to remove operand instances or keep them. 
Red Hat Docs

Remove the label kueue.openshift.io/managed=true from any namespaces if you’re removing Kueue.

Remove any ClusterQueues / LocalQueues / ResourceFlavors you created.

