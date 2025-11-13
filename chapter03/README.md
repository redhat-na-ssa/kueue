# Working with GPUs

## 1) Install GPU Operator (OperatorHub)

OpenShift Web Console → Operators → OperatorHub → search NVIDIA GPU Operator → Install (All namespaces is fine).
Accept defaults so it also deploys Node Feature Discovery and the NVIDIA device plugin.

CLI check:
```
oc get pods -n nvidia-gpu-operator
oc get ds -A | grep nvidia-device-plugin
```
## 2) Verify GPUs are allocatable
```
oc get nodes -o custom-columns='NAME:.metadata.name,GPUs:.status.allocatable.nvidia\.com/gpu'

```

You should see 1 (or more) for each of your two GPU nodes.

If not, wait for the operator to finish, or check the GPU Operator / NFD pods.

## 3) Label (and optionally taint) the two GPU nodes

Pick simple labels—here I’ll use gpu=node-a and gpu=node-b:

Replace with your node names:
```
NODE_A=<gpu-node-1>
NODE_B=<gpu-node-2>

oc label node $NODE_A gpu=node-a --overwrite
oc label node $NODE_B gpu=node-b --overwrite
```

(Optional) If you want these nodes dedicated to GPU workloads:
```
oc adm taint nodes $NODE_A dedicated=gpu:NoSchedule
oc adm taint nodes $NODE_B dedicated=gpu:NoSchedule
```

(If you taint them, we’ll add matching tolerations in the flavors so Kueue can steer admitted pods there.)

## 4) Create ResourceFlavors for those nodes

Save as 01-rf-gpu.yaml:
```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ResourceFlavor
metadata:
  name: gpu-flavor  
spec:
  nodeLabels:
    nvidia.com/gpu.present: "true"
  tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
```

Apply:
```
oc apply -f 01-rf-gpu.yaml
```
## 5) Create a ClusterQueue with GPU quota per flavor

Assuming each node has 1 GPU (adjust nominalQuota if different).

02-cq-gpu.yaml:
```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata:
  name: gpu-cluster-queue
spec:
  namespaceSelector: {}   # allow all namespaces (or restrict via a selector)
  resourceGroups:
    - coveredResources: 
      - "nvidia.com/gpu"
      flavors:
        - name: gpu-flavor
          resources:
            - name: "nvidia.com/gpu"
              nominalQuota: 1
```

Apply:
```
oc apply -f 02-cq-gpu.yaml
```

Note: You can also add CPU/memory to coveredResources if you want Kueue to account them too. For many setups, only accounting GPUs is plenty—OpenShift’s default scheduler will handle CPU/mem once Kueue admits the workload and injects node affinity/tolerations.

## 6) Create a Project and LocalQueue
```
oc new-project ml-demo
```

03-lq-ml-demo.yaml:
```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: LocalQueue
metadata:
  name: gpu-queue
  namespace: kueue-demo
spec:
  clusterQueue: gpu-cluster-queue
```

Apply:
```
oc apply -f 03-lq-ml-demo.yaml
```
## 7) Submit GPU Jobs (OpenShift-friendly examples)
A) Single-GPU job (1 pod uses 1 GPU on either node)

Use a CUDA base that runs fine under OpenShift’s default SCC. The standard NVIDIA CUDA images work; no privileged needed.

job-gpu-1.yaml:
~~~yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: demo-gpu-1
  namespace: kueue-demo
  labels:
    kueue.x-k8s.io/queue-name: gpu-queue
spec:
  parallelism: 1
  completions: 1
  template:
    spec:
      tolerations:  
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
      restartPolicy: Never
      containers:
        - name: main
          image: nvidia/cuda:12.3.1-base-ubuntu22.04
          command: ["bash", "-lc", "nvidia-smi || echo 'no nvidia-smi'; sleep 20"]
          resources:
            limits:
              nvidia.com/gpu: 1
~~~
```
oc apply -f 04-job-01.yaml
```
B) Two GPUs across both nodes (2 pods × 1 GPU each)

If you want to consume both nodes at once, run two pods that each request a single GPU:

05-job-gpu-2pods.yaml:
~~~yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: demo-gpu-2pods
  namespace: kueue-demo
  labels:
    kueue.x-k8s.io/queue-name: gpu-queue
spec:
  parallelism: 2
  completions: 2
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: main
          image: nvidia/cuda:12.3.1-base-ubuntu22.04
          command: ["bash", "-lc", "nvidia-smi || echo 'no nvidia-smi'; sleep 20"]
          resources:
            limits:
              nvidia.com/gpu: 1
~~~
```
oc apply 05-job-cpu-2pods.yaml