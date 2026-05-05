# Triton Model Setup for OpenShift

This guide covers how to download and load NGC models into the router-server for Triton-based routing on OpenShift.

## Overview

By default, the Helm chart deploys the router-server with:
- A **ReadWriteOnce (RWO)** PVC mounted at `/model_repository`
- An **empty model repository** -- Triton Inference Server starts but has no models loaded
- The `tritonserver` command launched directly via the Helm template's `command` field

**You must populate the model repository before Triton routing will function.** This guide covers two approaches:

| Aspect | Option A: Download Pod | Option B: Custom Image |
|--------|----------------------|----------------------|
| **PVC Access Mode** | ReadWriteMany (RWX) required | ReadWriteOnce (default) |
| **Storage Class** | CephFS (`ocs-storagecluster-cephfs`) | Any (e.g., `gp3-csi`) |
| **Startup Time** | Fast (models pre-loaded on PVC) | Slower on first start (~5-10 min download) |
| **Image Modification** | None | Custom Dockerfile required |
| **Helm Template Modification** | None | Required (local only, not committed upstream) |
| **Model Persistence** | On PVC, survives pod restarts | On PVC, downloaded once then skipped |

## Prerequisites

Before proceeding with either option, ensure:
- Triton routing is deployed (`routerServer.enabled=true`)
- The NGC CLI API key secret exists:
  ```bash
  oc create secret generic ngc-cli-key \
    --from-literal=ngc_cli_api_key="$NGC_API_KEY"
  ```
- The Helm-created PVC exists:
  ```bash
  oc get pvc llm-router-router-server-model-repo
  ```

---

## Option A: Download Pod with RWX PVC

This approach uses a separate Kubernetes pod to download models from NGC directly into the PVC. Since both the download pod and the router-server pod need concurrent access to the same PVC, it must be **ReadWriteMany (RWX)**.

### Step 1: Reconfigure PVC for ReadWriteMany

The default Helm deployment creates an RWO PVC. To use a download pod, you need to switch to RWX. Since PVC access modes are **immutable**, you must delete the existing PVC and redeploy:

```bash
# Delete the existing RWO PVC
oc delete pvc llm-router-router-server-model-repo

# Redeploy with RWX and CephFS storage class
helm upgrade llm-router ./deploy/helm/llm-router \
  --reuse-values \
  --set routerServer.volumes.modelRepository.storage.persistentVolumeClaim.accessMode=ReadWriteMany \
  --set routerServer.volumes.modelRepository.storage.persistentVolumeClaim.storageClass=ocs-storagecluster-cephfs

# Verify the new PVC is RWX and bound
oc get pvc llm-router-router-server-model-repo -o custom-columns=NAME:.metadata.name,ACCESS:.spec.accessModes,STATUS:.status.phase --no-headers
# Expected: llm-router-router-server-model-repo   [ReadWriteMany]   Bound
```

### Step 2: Create Download ConfigMap

```bash
# Ensure you're in the project root directory containing the Makefile
oc create configmap download-makefile --from-file=Makefile

# Verify Makefile was added correctly
oc describe configmap download-makefile | grep -A5 "Data"
```

### Step 3: Create and Run Download Pod

Save the following as `download-models.yaml` and apply it:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ngc-model-downloader
  namespace: your-namespace  # Replace with your namespace
spec:
  restartPolicy: Never
  securityContext:
    runAsNonRoot: true
  containers:
  - name: downloader
    image: registry.redhat.io/rhel8/python-39:latest
    command: ["/bin/bash"]
    args:
    - -c
    - |
      echo "Installing system dependencies..."
      yum install -y curl wget unzip libxcrypt-compat

      echo "Creating routers directory..."
      cd /workspace
      mkdir -p routers

      echo "Running make download..."
      export NGC_CLI_API_KEY="$NGC_API_KEY"
      export NGC_CLI_ORG="nvidia/nemo"
      make download

      echo "Download completed. Setting up model repository..."

      # Create model repository structure
      mkdir -p /model_repository/task_router/1
      mkdir -p /model_repository/complexity_router/1
      mkdir -p /model_repository/task_router_ensemble/1
      mkdir -p /model_repository/complexity_router_ensemble/1
      mkdir -p /model_repository/preprocessing_task_router/1
      mkdir -p /model_repository/postprocessing_task_router/1
      mkdir -p /model_repository/preprocessing_complexity_router/1
      mkdir -p /model_repository/postprocessing_complexity_router/1

      # Copy models (~700MB each)
      cp -v routers/task_router/1/model.pt /model_repository/task_router/1/model.pt
      cp -v routers/complexity_router/1/model.pt /model_repository/complexity_router/1/model.pt

      # Copy configuration and Python files
      for py_file in $(find routers/ -name "*.py"); do
        rel_path=$(echo "$py_file" | sed 's|routers/||')
        dest_dir="/model_repository/$(dirname "$rel_path")"
        cp -v "$py_file" "$dest_dir/" || true
      done

      for config_file in $(find routers/ -name "config.pbtxt"); do
        rel_path=$(echo "$config_file" | sed 's|routers/||')
        dest_dir="/model_repository/$(dirname "$rel_path")"
        cp -v "$config_file" "$dest_dir/" || true
      done

      echo "Models downloaded directly to PVC"
      find /model_repository -name "*.pt" -exec ls -lh {} \;
    env:
    - name: NGC_API_KEY
      valueFrom:
        secretKeyRef:
          name: ngc-cli-key
          key: ngc_cli_api_key
    volumeMounts:
    - name: workspace
      mountPath: /workspace
    - name: makefile-volume
      mountPath: /workspace/Makefile
      subPath: Makefile
    - name: model-repository
      mountPath: /model_repository
    workingDir: /workspace
    resources:
      requests:
        memory: "2Gi"
        cpu: "1"
      limits:
        memory: "4Gi"
        cpu: "2"
  volumes:
  - name: workspace
    emptyDir:
      sizeLimit: 10Gi
  - name: makefile-volume
    configMap:
      name: download-makefile
  - name: model-repository
    persistentVolumeClaim:
      claimName: llm-router-router-server-model-repo
```

```bash
# Apply the download pod
oc apply -f download-models.yaml

# Monitor progress
oc logs -f ngc-model-downloader

# Wait for completion
oc wait --for=jsonpath='{.status.phase}'=Succeeded pod/ngc-model-downloader --timeout=600s

# Restart router-server to load the downloaded models
oc rollout restart deployment/llm-router-router-server

# Verify models loaded successfully
oc logs deployment/llm-router-router-server | grep "successfully loaded"
```

### RHOAI Notebook Alternative

For users with RHOAI workbench environments:

1. **Configure Workbench**: Ensure your RHOAI workbench has the same RWX PVC attached that the LLM Router router-server uses for model storage
2. **Run Deployment Notebook**: Execute [launchable/1_Deploy_LLM_Router.ipynb](../launchable/1_Deploy_LLM_Router.ipynb) from your RHOAI workbench
3. **Verify Shared Storage**: The notebook will download models to the shared PVC, making them available to the LLM Router deployment

> **Note**: This method leverages RHOAI's notebook environment while ensuring models are downloaded to the shared storage accessible by the LLM Router pods.

---

## Option B: Custom Docker Image with Runtime Download

This approach builds a custom router-server image that includes the NGC CLI and downloads models at container startup. It works with the default **ReadWriteOnce** PVC since both the download and inference happen within the same pod.

### Step 1: Create the Custom Dockerfile

Save the following as `router-server-with-models.dockerfile`:

```dockerfile
FROM nvcr.io/nvidia/tritonserver:24.10-py3

# Copy requirements and install Python dependencies
COPY src/router-server/requirements.txt /tmp/requirements.txt
RUN python3 -m pip install --upgrade pip --root-user-action=ignore
RUN pip install -r /tmp/requirements.txt --root-user-action=ignore

# Create model repository directory
RUN mkdir -p /model_repository

# Install NGC CLI
RUN cd /tmp && \
    wget --content-disposition https://api.ngc.nvidia.com/v2/resources/nvidia/ngc-apps/ngc_cli/versions/3.58.0/files/ngccli_linux.zip -O ngccli_linux.zip && \
    unzip -o ngccli_linux.zip && \
    mv ngc-cli /opt/ngc-cli && \
    rm ngccli_linux.zip

# Create a startup script that downloads models at runtime
RUN echo '#!/bin/bash' > /download_models.sh && \
    echo 'set -e' >> /download_models.sh && \
    echo '' >> /download_models.sh && \
    echo 'echo "Checking for existing models..."' >> /download_models.sh && \
    echo 'if [ -d "/model_repository/task_router_ensemble" ] && [ -d "/model_repository/complexity_router_ensemble" ]; then' >> /download_models.sh && \
    echo '    echo "Models already exist, skipping download"' >> /download_models.sh && \
    echo '    exit 0' >> /download_models.sh && \
    echo 'fi' >> /download_models.sh && \
    echo '' >> /download_models.sh && \
    echo 'echo "Models not found, downloading..."' >> /download_models.sh && \
    echo '' >> /download_models.sh && \
    echo 'if [ -z "$NGC_CLI_API_KEY" ]; then' >> /download_models.sh && \
    echo '    echo "NGC_CLI_API_KEY environment variable is required"' >> /download_models.sh && \
    echo '    exit 1' >> /download_models.sh && \
    echo 'fi' >> /download_models.sh && \
    echo '' >> /download_models.sh && \
    echo 'cd /tmp' >> /download_models.sh && \
    echo 'export NGC_CLI_ORG="nvidia/nemo"' >> /download_models.sh && \
    echo '' >> /download_models.sh && \
    echo 'echo "Downloading task router model..."' >> /download_models.sh && \
    echo '/opt/ngc-cli/ngc registry model download-version "nvidia/nemo/prompt-task-and-complexity-classifier:task-llm-router"' >> /download_models.sh && \
    echo '' >> /download_models.sh && \
    echo 'echo "Downloading complexity router model..."' >> /download_models.sh && \
    echo '/opt/ngc-cli/ngc registry model download-version "nvidia/nemo/prompt-task-and-complexity-classifier:complexity-llm-router"' >> /download_models.sh && \
    echo '' >> /download_models.sh && \
    echo 'echo "Installing models..."' >> /download_models.sh && \
    echo 'cp -r prompt-task-and-complexity-classifier_vtask-llm-router/* /model_repository/' >> /download_models.sh && \
    echo 'cp -r prompt-task-and-complexity-classifier_vcomplexity-llm-router/* /model_repository/' >> /download_models.sh && \
    echo '' >> /download_models.sh && \
    echo 'rm -rf prompt-task-and-complexity-classifier_*' >> /download_models.sh && \
    echo '' >> /download_models.sh && \
    echo 'echo "Model download complete!"' >> /download_models.sh && \
    chmod +x /download_models.sh

# Create entrypoint script
RUN echo '#!/bin/bash' > /entrypoint.sh && \
    echo 'set -e' >> /entrypoint.sh && \
    echo '' >> /entrypoint.sh && \
    echo 'echo "Starting Triton server with model download..."' >> /entrypoint.sh && \
    echo '' >> /entrypoint.sh && \
    echo '/download_models.sh' >> /entrypoint.sh && \
    echo '' >> /entrypoint.sh && \
    echo 'exec tritonserver --log-verbose=1 --model-repository=/model_repository "$@"' >> /entrypoint.sh && \
    chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
```

### Step 2: Build and Push the Image

```bash
podman build --platform=linux/amd64 \
  -f router-server-with-models.dockerfile \
  -t your-registry.com/router-server-with-models:your-tag .

podman push your-registry.com/router-server-with-models:your-tag
```

### Step 3: Modify the Helm Deployment Template (Local Only)

> **Important**: The standard Helm deployment template uses a `command` field that launches `tritonserver` directly, which **overrides** the custom image's `ENTRYPOINT`. You must modify the template locally to let the image's entrypoint handle model download and server startup. This modification is **not committed upstream**.

Apply the following changes to `deploy/helm/llm-router/templates/router-server-deployment.yaml`:

**Change 1: Replace `command` with `args`**

Remove the `command` block and replace it with `args` (without the `tritonserver` binary, since the image's entrypoint already calls it):

```yaml
# BEFORE (original):
          command:
            - "tritonserver"
            - "--log-verbose=1"
            - "--exit-on-error=false"
            - "--allow-http=true"
            - "--allow-grpc=true"
            - "--allow-metrics=true"
            {{- if .Values.routerServer.modelRepository.path }}
            - "--model-repository={{ .Values.routerServer.modelRepository.path }}"
            {{- end }}

# AFTER (modified):
          args:
            - "--exit-on-error=false"
            - "--allow-http=true"
            - "--allow-grpc=true"
            - "--allow-metrics=true"
```

**Change 2: Add NGC secret environment variable**

Add the following block inside the `env:` section, after the `routerServer.env` block:

```yaml
            {{- if .Values.routerServer.ngcSecret }}
            - name: NGC_CLI_API_KEY
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.routerServer.ngcSecret.secretName | default "ngc-cli-key" }}
                  key: {{ .Values.routerServer.ngcSecret.key | default "ngc_cli_api_key" }}
            {{- end }}
```

**Change 3: Make the `env:` block unconditional**

The original template conditionally renders the `env:` block only when cloud auth or `routerServer.env` is set. Remove the outer condition so it always renders:

```yaml
# BEFORE (original):
          {{- if or .Values.routerServer.modelRepository.cloudAuth.aws.enabled ... .Values.routerServer.env }}
          env:
            ...
          {{- end }}

# AFTER (modified):
          env:
            ...
          # (remove the outer {{- if or ... }} and its closing {{- end }})
```

### Step 4: Create NGC Secret and Deploy

```bash
# Create NGC CLI key secret (if not already created)
oc create secret generic ngc-cli-key \
  --from-literal=ngc_cli_api_key="$NGC_API_KEY"

# Deploy (or upgrade) with the custom image and NGC secret enabled
helm install llm-router ./deploy/helm/llm-router \
  --set openshift.enabled=true \
  --set app.enabled=true \
  --set routerServer.enabled=true \
  --set routerController.enabled=true \
  --set global.imageRegistry=your-registry.com/namespace/ \
  --set routerServer.image.repository=router-server-with-models \
  --set routerServer.image.tag=your-tag \
  --set routerServer.ngcSecret.enabled=true \
  --set routerController.image.tag=your-tag \
  --set app.image.tag=your-tag \
  --set 'routerServer.env[0].name=HF_DATASETS_CACHE' \
  --set 'routerServer.env[0].value=/tmp/hf_cache' \
  --set 'routerServer.env[1].name=HUGGINGFACE_HUB_CACHE' \
  --set 'routerServer.env[1].value=/tmp/hf_cache' \
  --set 'routerServer.env[2].name=TRANSFORMERS_CACHE' \
  --set 'routerServer.env[2].value=/tmp/hf_cache' \
  --set 'routerServer.env[3].name=HOME' \
  --set 'routerServer.env[3].value=/tmp'
```

### Step 5: Verify Model Loading

```bash
# Watch the router-server logs for model download and loading
oc logs -f deployment/llm-router-router-server

# Expected output sequence:
# 1. "Starting Triton server with model download..."
# 2. "Checking for existing models..."
# 3. "Models not found, downloading..."
# 4. "Downloading task router model..."
# 5. "Downloading complexity router model..."
# 6. "Model download complete!"
# 7. Triton model loading messages

# On subsequent restarts, models are found on the PVC and download is skipped:
# "Models already exist, skipping download"
```

---

## Troubleshooting

### Model Repository Issues

If Triton routing fails to load models:

```bash
# Verify repository structure
oc exec deployment/llm-router-router-server -- find /model_repository -type f | head -20

# Check for actual model files (.pt files, ~700MB each)
oc exec deployment/llm-router-router-server -- find /model_repository -name "*.pt" -exec ls -lh {} \;

# Check Triton server logs for loading errors
oc logs deployment/llm-router-router-server | grep -E "model|error|fail"
```

**Common Issues:**

| Problem | Cause | Solution |
|---------|-------|----------|
| Empty model repository (`lost+found` only) | Models not downloaded | Follow Option A or B above |
| `NGC_CLI_API_KEY environment variable is required` | Missing NGC secret | Create `ngc-cli-key` secret (see Prerequisites) |
| `Missing org` error from NGC CLI | `NGC_CLI_ORG` not set | Ensure the download script exports `NGC_CLI_ORG="nvidia/nemo"` |
| `ngccli` pip install fails | Dockerfile uses wrong NGC CLI installation | Use the Dockerfile in Option B (installs NGC CLI from official zip) |
| Triton starts but entrypoint doesn't run | Helm `command` overrides image `ENTRYPOINT` | Apply the template changes in Option B Step 3 |
| Cache permission errors (HuggingFace) | OpenShift security context blocks root filesystem writes | Set HF cache env vars to `/tmp/hf_cache` (see [main guide](openshift.md#cache-configuration-required-for-triton-routing-only)) |
| Insufficient storage | PVC too small for ~1.5GB of model files | Ensure PVC is at least 100Gi |

### PVC Access Mode Issues

PVC access modes are **immutable** -- you cannot change an existing PVC's access mode. If you need to switch between RWO and RWX:

```bash
# Delete the existing PVC (this will delete any downloaded models)
oc delete pvc llm-router-router-server-model-repo

# Redeploy with the desired access mode
helm upgrade llm-router ./deploy/helm/llm-router \
  --reuse-values \
  --set routerServer.volumes.modelRepository.storage.persistentVolumeClaim.accessMode=ReadWriteMany \
  --set routerServer.volumes.modelRepository.storage.persistentVolumeClaim.storageClass=ocs-storagecluster-cephfs
```

> **Recommendation**: If you are unsure which option to use, start with **Option A** (download pod) for simplicity. Use **Option B** (custom image) if you need to avoid RWX storage or want a self-contained deployment.

Last updated: May 2026
