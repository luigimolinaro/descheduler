# Using Kubernetes Descheduler on Cloud Pak for Data 5.1.3

This document provides a detailed walkthrough on how to install, configure, and run the Kubernetes descheduler on a Cloud Pak for Data 5.1.3 environment. It also includes a performance analysis before and after running the descheduler to demonstrate its impact on cluster resource utilization.

---

## Prerequisites

- OpenShift cluster with administrative access (`oc` CLI with cluster-admin rights)
- Helm 3 installed locally
- Access to the Cloud Pak for Data namespace and nodes
- Basic familiarity with Kubernetes/OpenShift concepts

---

## 1. Installing Helm 3

If Helm is not installed, follow these steps to install Helm 3:

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm version
````

---

## 2. Setting up the Descheduler Namespace and Service Account

Create a dedicated namespace for descheduler:

```bash
oc create namespace descheduler
```

Create the service account and assign appropriate SCC permissions:

```bash
oc create serviceaccount descheduler -n descheduler
oc adm policy add-scc-to-user anyuid -z descheduler -n descheduler
```

---

## 3. Adding the Descheduler Helm Repository and Installing

Add the official descheduler Helm repo:

```bash
helm repo add descheduler https://kubernetes-sigs.github.io/descheduler/
helm repo update
```

Create a `values.yaml` file to customize the descheduler configuration if needed. (Or just wait 2 minute to let job run) For example:

```yaml
apiVersion: descheduler/v1alpha1
kind: DeschedulerPolicy
strategies:
  "RemoveDuplicates":
    enabled: true
  "LowNodeUtilization":
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

Install descheduler using Helm:

```bash
helm install descheduler descheduler/descheduler --namespace descheduler -f values.yaml
```

Verify the installation:

```bash
helm list -n descheduler
oc get pods -n descheduler
```

---

## 4. Running the Descheduler Job

To trigger descheduler actions, create a `job.yaml` that runs descheduler as a job, for example:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: descheduler-job
  namespace: descheduler
spec:
  template:
    spec:
      serviceAccountName: descheduler
      containers:
      - name: descheduler
        image: k8s.gcr.io/descheduler/descheduler:v0.25.0
        command:
        - /bin/descheduler
        - --policy-config-file=/policy.yaml
        - --v=2
        volumeMounts:
        - mountPath: /policy.yaml
          name: policy
          subPath: policy.yaml
      restartPolicy: Never
      volumes:
      - name: policy
        configMap:
          name: descheduler-policy
  backoffLimit: 4
```

Apply it with:

```bash
oc apply -f job.yaml
```

Check job status and logs:

```bash
oc get jobs -n descheduler
oc logs job/descheduler-job -n descheduler
```

---

## 5. Initial Resource Usage Before Running Descheduler

Before running descheduler, the resource usage across nodes was unbalanced, as shown below:

| Node    | CPU (cores) | CPU % | Memory (Mi) | Memory % |
| ------- | ----------- | ----- | ----------- | -------- |
| master0 | 2200m       | 29%   | 12000Mi     | 82%      |
| master1 | 1500m       | 20%   | 13000Mi     | 89%      |
| master2 | 1800m       | 25%   | 11000Mi     | 75%      |
| worker0 | 5000m       | 32%   | 21000Mi     | 45%      |
| worker1 | 3500m       | 18%   | 24000Mi     | 50%      |
| worker2 | 1000m       | 8%    | 5000Mi      | 12%      |
| worker3 | 900m        | 7%    | 4000Mi      | 10%      |

---

## 6. Resource Usage After Running Descheduler

After descheduler executed, workloads redistributed more evenly across the cluster:

| Node    | CPU (cores) | CPU % | Memory (Mi) | Memory % |
| ------- | ----------- | ----- | ----------- | -------- |
| master0 | 1402m       | 18%   | 9365Mi      | 64%      |
| master1 | 829m        | 11%   | 10749Mi     | 74%      |
| master2 | 722m        | 9%    | 7568Mi      | 52%      |
| worker0 | 3470m       | 22%   | 23785Mi     | 37%      |
| worker1 | 1680m       | 10%   | 26108Mi     | 42%      |
| worker2 | 1273m       | 8%    | 20286Mi     | 32%      |
| worker3 | 887m        | 5%    | 16602Mi     | 26%      |
| worker4 | 910m        | 5%    | 17572Mi     | 28%      |
| worker5 | 1537m       | 9%    | 6585Mi      | 10%      |

---

## 7. Conclusion

Using the Kubernetes descheduler on Cloud Pak for Data 5.1.3 significantly improved the cluster's resource utilization by balancing workloads across nodes. The descheduler helped reduce high memory pressure on master nodes and increased utilization on less busy workers, contributing to enhanced stability and performance.

This process included:

* Installing Helm 3 and adding the descheduler Helm repo
* Creating a dedicated namespace and assigning the necessary SCC permissions (`anyuid`) to the descheduler service account
* Deploying the descheduler with a custom policy to target duplicate pods and low node utilization scenarios
* Running the descheduler job and verifying pod evictions and rescheduling
* Monitoring node resource usage before and after to quantify the improvement

This approach can be adopted in similar Cloud Pak for Data or OpenShift environments to optimize workload distribution and cluster efficiency.
