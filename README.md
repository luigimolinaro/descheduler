Certo! Ecco tutto il contenuto formattato in **markdown** pronto per essere usato come `README.md`:

````markdown
# Descheduler Installation and Usage on Cloud Pak for Data 5.1.3

This document provides a detailed walkthrough on installing, configuring, and running Kubernetes Descheduler in an OpenShift environment hosting Cloud Pak for Data 5.1.3. The goal is to optimize pod distribution across cluster nodes by evicting pods from overloaded nodes and improving resource balancing.

---

## Prerequisites

- OpenShift 4.x cluster with administrative privileges (`oc` CLI access).
- Cloud Pak for Data 5.1.3 installed and running on the cluster.
- `helm` CLI installed locally or on bastion host.

---

## Step 1: Install Helm

If Helm is not installed, install Helm 3 using the official script:

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm version
````

---

## Step 2: Add Descheduler Helm Repository

Add the official descheduler Helm repo and update local cache:

```bash
helm repo add descheduler https://kubernetes-sigs.github.io/descheduler/
helm repo update
```

---

## Step 3: Create Namespace for Descheduler

Create a dedicated namespace to isolate descheduler resources:

```bash
oc create namespace descheduler
```

---

## Step 4: Configure Security Context Constraints (SCC)

Grant `anyuid` SCC to the descheduler service account for required privileges:

```bash
oc adm policy add-scc-to-user anyuid -z descheduler -n descheduler
```

---

## Step 5: Prepare Custom Values for Descheduler Helm Chart

Create a `values.yaml` file with your desired configuration. Example minimal configuration enabling key strategies for evicting pods on resource pressure:

```yaml
apiVersion: descheduler/v1alpha1
kind: DeschedulerPolicy
strategies:
  RemoveDuplicates:
    enabled: true
  LowNodeUtilization:
    enabled: true
    params:
      nodeResourceUtilizationThresholds:
        thresholds:
          cpu: 20
          memory: 20
        targetThresholds:
          cpu: 50
          memory: 50
```

---

## Step 6: Install Descheduler with Helm

Install or upgrade descheduler in the `descheduler` namespace using the prepared `values.yaml`:

```bash
helm install descheduler descheduler/descheduler --namespace descheduler -f values.yaml
# or, if upgrading:
helm upgrade descheduler descheduler/descheduler --namespace descheduler -f values.yaml
```

---

## Step 7: Verify Deployment

Check pods and logs to confirm descheduler is running:

```bash
oc get pods -n descheduler
oc logs -n descheduler -l app=descheduler
```

---

## Step 8: Running Descheduler Job Manually (Optional)

For finer control or periodic runs, create a Kubernetes job manifest or run descheduler as a standalone job, targeting specific nodes or using custom policies.

---

## Analysis of Results After Running Descheduler

Following the deployment and execution of the descheduler, resource usage across cluster nodes showed improved distribution. Initially, workloads were concentrated on fewer nodes causing high CPU and memory utilization spikes. After descheduler runs:

* CPU usage spread more evenly across master and worker nodes.
* Memory usage normalized with no single node showing critical pressure.
* Pod eviction based on duplicate pods and low node utilization strategies reduced hotspots.


Example metrics snapshot post-descheduler:

| Node                          | CPU Usage | Memory Usage |
| ----------------------------- | --------- | ------------ |
| master0.banco.cp.fyre.ibm.com | 18%       | 64%          |
| master1.banco.cp.fyre.ibm.com | 11%       | 74%          |
| master2.banco.cp.fyre.ibm.com | 9%        | 52%          |
| worker0.banco.cp.fyre.ibm.com | 22%       | 37%          |
| worker1.banco.cp.fyre.ibm.com | 10%       | 42%          |
| worker2.banco.cp.fyre.ibm.com | 8%        | 32%          |
| worker3.banco.cp.fyre.ibm.com | 5%        | 26%          |
| worker4.banco.cp.fyre.ibm.com | 5%        | 28%          |
| worker5.banco.cp.fyre.ibm.com | 9%        | 10%          |

This improved balancing ensures Cloud Pak for Data components run efficiently with lower risk of resource contention.

---

## Troubleshooting

* **Permission Issues:** Verify SCC binding (`anyuid`) and service account permissions.
* **Pod Eviction Failures:** Confirm pod disruption budgets allow eviction.
* **Helm Chart Version:** Always check compatibility with your Kubernetes/OpenShift version.

---

## References

* [Kubernetes Descheduler GitHub](https://github.com/kubernetes-sigs/descheduler)
* [Descheduler Helm Chart](https://github.com/kubernetes-sigs/descheduler/tree/master/charts/descheduler)
* [OpenShift SCC Documentation](https://docs.openshift.com/container-platform/latest/authentication/managing-security-context-constraints.html)

