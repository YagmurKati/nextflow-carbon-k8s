# Carbon-Aware Nextflow on Kubernetes

This project implements a **carbon-aware Nextflow workflow** running on **Kubernetes**. The pipeline is triggered **only when electricity carbon intensity is below a specified threshold**, using live data from [ElectricityMap](https://www.electricitymap.org/).

Jobs are launched dynamically by a Kubernetes **CronJob** that checks real-time carbon levels every hour and submits a Nextflow workflow if conditions are "green."

---

## 📦 Components Overview

| File | Description |
|------|-------------|
| `green_k8s_workflow.nf` | Main Nextflow DSL2 workflow |
| `nextflow.config` | Configuration for Kubernetes execution, pod labels, resources, etc. |
| `nextflow-job.yaml` | Templated Kubernetes Job (populated by carbon check script) |
| `check_carbon_and_submit.sh` | Script that checks carbon level and applies the job if below threshold |
| `carbon-check-cronjob.yaml` | Kubernetes CronJob to run the carbon-check script hourly |
| `nextflow-pvc.yaml` | RWX PersistentVolumeClaim used to store workflow data |
| `pvc-uploader.yaml` | Long-running pod used to upload files into the PVC manually |
| `nextflow-trigger-rbac.yaml` | RBAC config that permits CronJob to create jobs |
| `bootstrap-nextflow-carbon.sh` | Automated deployment script for the entire stack |

---

## ⚙️ Requirements

- Kubernetes cluster (single-node or multi-node)
- RWX (ReadWriteMany) StorageClass (e.g. CephFS, NFS)
- A namespace (e.g. `yagmur`)
- [ElectricityMap API Token](https://www.electricitymap.org/)
- `kubectl` CLI configured
- Access to a Docker registry (for the carbon-checker image)

---
## 🧭 Namespace Setup

This project assumes that you are working within a specific Kubernetes namespace.

### 🔍 How to Find Your Namespace

If your Kubernetes administrator has already assigned you a namespace, you can list all available namespaces with:

```bash
kubectl get namespaces
```
Look for a namespace that matches your team, username, or was communicated to you by your admin.

If you're unsure which namespace to use, ask your administrator.

## 📌 Using Your Namespace

Once you know your namespace, make sure to include the -n flag in your kubectl commands:
```bash
kubectl get pods -n your-namespace
```
Replace your-namespace with your actual namespace name.

    ⚠️ Note: Do not create your own namespace unless you’ve confirmed you have permission to do so.

If you do have permissions and want to create a namespace:
```bash
kubectl create namespace my-namespace
```

# 1. Set your Kubernetes namespace
export NAMESPACE=my-namespace

# 2. Bootstrap all resources (PVC, RBAC, CronJob, etc.)
chmod +x bootstrap-nextflow-carbon.sh
./bootstrap-nextflow-carbon.sh

---

## 🚀 Quick Start: One-Step Setup

Use the provided bootstrap script to deploy everything:

```bash
./bootstrap-nextflow-carbon.sh
```
This will:

    Create the yagmur namespace (if missing)

    Create the PVC

    Deploy the uploader pod

    Create the ConfigMap from your script and job template

    Apply RBAC permissions

    Deploy the CronJob that checks carbon intensity and submits a Nextflow job

## 📤 Upload Workflow Files

Once the pvc-uploader pod is running:
```bash
kubectl cp green_k8s_workflow.nf yagmur/pvc-uploader:/workspace/
kubectl cp nextflow.config yagmur/pvc-uploader:/workspace/
kubectl cp nextflow-job.yaml yagmur/pvc-uploader:/workspace/
```
## 🧪 Trigger a Manual Test (Optional)

To test job submission without waiting for the carbon threshold:
```bash
kubectl create job --from=cronjob/carbon-checker carbon-checker-test-run -n yagmur
kubectl logs -f job/carbon-checker-test-run -n yagmur
```
## 🔍 Monitor Execution

List current jobs and pods:
```bash
kubectl get jobs -n yagmur
kubectl get pods -n yagmur
```
Get logs from the latest Nextflow job (replace with actual job name):
```bash
kubectl logs -f job/nextflow-run-<timestamp> -n yagmur
```
## 📥 Retrieve Output

List files in PVC:
```bash
kubectl exec -it pvc-uploader -n yagmur -- ls /workspace
```
Download output artifacts:
```bash
kubectl cp yagmur/pvc-uploader:/workspace/report.html ./report.html
kubectl cp yagmur/pvc-uploader:/workspace/timeline.html ./timeline.html
kubectl cp yagmur/pvc-uploader:/workspace/trace.txt ./trace.txt
```
## 🧠 Carbon Logic (What Makes This "Green"?)

The script check_carbon_and_submit.sh uses the ElectricityMap API to fetch live carbon intensity values:
```bash
carbon=$(curl -s "https://api.electricitymap.org/v3/carbon-intensity/latest?zone=$ZONE" \
  -H "auth-token: $AUTH_TOKEN" | jq -r '.carbonIntensity')
```
If the value is below a defined threshold (e.g. 350 gCO₂/kWh), the Nextflow job is created via:
```bash
envsubst < /scripts/nextflow-job.yaml > /tmp/rendered-job.yaml
kubectl apply -f /tmp/rendered-job.yaml
```
Otherwise, it exits and waits for the next CronJob interval.

## ⚙️ Runtime Config

Defined in nextflow.config:

    - Executor: k8s

    - Node selectors: Use labels like node-type=highcpu, node-type=highmem, node-type=gpu

    - Containers: All processes use ubuntu:22.04 unless changed

    - Output: Generates trace.txt, report.html, timeline.html

## 🧩 Customization Ideas

    - 📊 Log carbon data to a persistent CSV for plotting

    - 🗃  Organize each run under /workspace/results/<timestamp>

    - 🔔 Notify Slack/email when green jobs are launched

    - 📈 Visualize carbon-aware scheduling with Grafana

    - ☁ Push outputs to object storage (e.g., S3, GCS)

## 🛡  Security Notes

    - The default service account is used with limited RBAC (job creation only).
    - Make sure your API token is not hardcoded in production environments.
    - Use Secret objects for production deployments.

## 🙋 FAQ

Q: What happens if carbon is too high?
A: Nothing is scheduled. The CronJob retries every hour by default.

Q: Can I change the region?
A: Yes! Update ZONE="DE" in check_carbon_and_submit.sh to your ElectricityMap zone (e.g. "FR").

Q: Does this support GPUs?
A: Yes. GPU-labeled processes (gpu_k8s) are defined but disabled by default. You can enable them in the workflow.
