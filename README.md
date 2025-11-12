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


Please visit the chapters to further explore.