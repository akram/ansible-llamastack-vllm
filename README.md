# Ansible automation for Llama Stack operator, GPU setup, and standalone vLLM on OpenShift

This project automates the following steps:

0. **Install Red Hat OpenShift AI (RHOAI)**
1. **Install Llama Stack Operator**
2. **Install NFD & NVIDIA GPU Operator and verify GPU availability**
3. **Deploy a standalone vLLM server, expose it with a Route, and smoke‑test it**
4. **Deploy LlamaStack Distribution with vLLM and Milvus DB**

> **Assumptions**
> - You already have an OpenShift/ROSA cluster and are logged in (`oc whoami`).  
> - Your workstation has Ansible 2.14+ and Python packages required by `kubernetes.core`.
> - You have a valid kubeconfig and enough permissions.
> - You have at least one GPU node in the cluster (e.g., via a GPU machine pool).

## Quick start

```bash
# 0) Install collection dependency
ansible-galaxy collection install -r requirements.yml

# 1) Export kubeconfig if not standard
export KUBECONFIG=~/.kube/config

# 2) Provide your HF token (or set it in group_vars/all.yml)
export HF_TOKEN=hf_xxx

# 3) Run everything end-to-end
ansible-playbook site.yml
```

You can also run step-by-step:

```bash
ansible-playbook playbooks/00_install_rhoai.yml
ansible-playbook playbooks/01_install_llama_operator.yml
ansible-playbook playbooks/02_configure_gpus.yml
ansible-playbook playbooks/03_deploy_vllm.yml
ansible-playbook playbooks/04_deploy_llamastack_distribution.yml
```

## Configuration

Everything is in `group_vars/all.yml`. Key knobs:

- `rhoai.*` – Red Hat OpenShift AI operator install
  - `namespace` – RHOAI operator namespace (default: "redhat-ods-operator")
  - `channel` – Operator channel (default: "stable")
  - `dsc_name` – DataScienceCluster instance name (default: "default-dsc")
- `llamastack_operator_url` – upstream operator manifest
- `nfd.*` – Node Feature Discovery operator install
- `gpu_operator.*` – NVIDIA GPU Operator install
  - Leave `channel` empty to auto-detect defaultChannel from the PackageManifest
  - `driver_install_wait_minutes` – Time to wait for NVIDIA driver installation (default: 15 minutes)
  - `skip_driver_wait` – Set to `true` to skip waiting for driver installation (default: `false`)
- `llamastack_ns` – namespace used for vLLM resources (`llamastack`)
- `hf_token` / env `HF_TOKEN` – your HF access token
- `llama_template_url` – chat template ConfigMap (pulled and applied as-is)
- `vllm_image`, `vllm_model`, `vllm_port`, `vllm_dtype`, `vllm_max_model_len`, `vllm_gpu_count`
- `llamastack_distribution.*` – LlamaStack Distribution configuration
  - `name` – Distribution name (default: "llama-test-milvus")
  - `image` – LlamaStack image (default: "quay.io/opendatahub/llama-stack:odh")
  - `inference_model` – Model for inference (default: "meta-llama/Llama-3.2-3B-Instruct")
  - `port` – Service port (default: 8321)
  - `milvus_db_path` – Milvus database path
  - `fms_orchestrator_url` – FMS Orchestrator URL for TrustyAI integration

## What the playbooks do

### Step 0 – Red Hat OpenShift AI (RHOAI)
- Installs the **RHOAI operator** from `redhat-operators` catalog
- Creates a **DataScienceCluster** instance with all components enabled:
  - CodeFlare - Multi-cluster compute orchestration
  - Dashboard - Web-based user interface  
  - Data Science Pipelines - ML workflow orchestration
  - KServe - Model serving platform
  - ModelMesh - Multi-model serving
  - Ray - Distributed computing framework
  - Workbenches - Jupyter notebook environments
- Waits for the DataScienceCluster to be ready

### Step 1 – Llama Stack Operator
- Applies `operator.yaml` from the official repo and waits for the operator deployment to be available.

### Step 2 – GPUs
- Installs **Node Feature Discovery** (from `redhat-operators`) and creates a default `NodeFeatureDiscovery` CR.
- Installs **NVIDIA GPU Operator** (from `certified-operators`) and creates a default `ClusterPolicy`.
- **Waits for NVIDIA driver installation** (configurable via `gpu_operator.driver_install_wait_minutes`)
  - The driver is compiled for your specific OpenShift kernel using OpenShift Driver Toolkit
  - This process typically takes 10-15 minutes depending on your instance type
  - The playbook actively polls the ClusterPolicy status every 30 seconds until ready or timeout
  - You can monitor progress with: `kubectl get pods -n nvidia-gpu-operator -w`
- Verifies that at least one node exposes `nvidia.com/gpu` capacity.
- Warns if the NFD GPU PCI label is not found (you might still be waiting for NFD or need a GPU pool).

### Step 3 – vLLM
- Creates the `llamastack` namespace and a `hf-token-secret` containing your HF token.
- Downloads and applies the upstream chat template ConfigMap.
- Deploys vLLM with the given model (defaults to `meta-llama/Llama-3.2-3B-Instruct`) and 1 GPU.
- Creates a `Service` and an OpenShift `Route`.
- Performs a simple `/v1/chat/completions` POST to confirm the endpoint responds.

### Step 4 – LlamaStack Distribution with vLLM and Milvus DB
- Creates a **LlamaStackDistribution** CR with vLLM integration
- Configures **Milvus inline DB** for vector storage
- Uses the **OpenDataHub LlamaStack image** (`quay.io/opendatahub/llama-stack:odh`)
- Connects to the vLLM service from Step 3
- Exposes the distribution via **OpenShift Route**
- Provides monitoring and access information

## Notes & troubleshooting

- If operator channels differ on your cluster, override them in `group_vars/all.yml`.
- If you don't see GPUs, confirm the machine pool type (e.g., `g4dn.xlarge`) and give the GPU operator time to finish provisioning.
- **GPU Driver Installation**: The NVIDIA driver compilation can take 10-15 minutes. If you want to skip the wait:
  - Set `gpu_operator.skip_driver_wait: true` in `group_vars/all.yml`
  - Or set `gpu_operator.driver_install_wait_minutes: 0` to skip the pause
  - Monitor manually: `kubectl get clusterpolicy gpu-cluster-policy -o yaml`
- For KServe-based vLLM (instead of standalone), skip Step 3 here and follow your KServe instructions.

---

This repo encodes commands equivalent to the original instructions (`oc apply …`, `oc expose …`, etc.) using `kubernetes.core.k8s` primitives so it’s idempotent and repeatable.
