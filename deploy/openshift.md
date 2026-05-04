# NVIDIA LLM Router - OpenShift Deployment Guide

This guide explains how to deploy the NVIDIA LLM Router on OpenShift using the provided Helm chart with OpenShift-specific configurations.

## Prerequisites

### Common Requirements (All Deployments)
- **OpenShift Cluster**: 4.10+ with minimum 3 worker nodes (4 vCPU, 16GB RAM each)
- **Helm**: 3.2.0+ installed and configured
- **NVIDIA API Key**: From [NVIDIA API Catalog](https://build.nvidia.com/settings/api-keys)
  - Format: `nvapi-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` (47 characters)
  - Required scopes: Model access for routing
- **NGC API Key**: From [NGC Portal](https://org.ngc.nvidia.com/setup/api-keys) 
  - Format: 40+ alphanumeric characters
  - Required for: Container registry access and model downloads
- **Container Images**: Pre-built images for router-controller and app components
  - Must be pushed to accessible container registry
  - Architecture: linux/amd64 (OpenShift standard)
- **Network Access**: Outbound HTTPS to `api.nvidia.com`, `build.nvidia.com`, `api.ngc.nvidia.com`

### Additional Requirements (Triton Routing Only)
- **GPU Hardware**: Nodes with 16GB+ VRAM (Tesla T4, V100, A10, A100, etc.)
- **NVIDIA GPU Operator**: Installed and validated on OpenShift cluster
- **Storage**: ReadWriteMany (RWX) storage class (CephFS, NFS, etc.)
- **Network**: Egress access to api.ngc.nvidia.com and build.nvidia.com
- **Container Images**: router-server component built for your architecture
- **GPU Node Configuration**: Tolerations for tainted GPU nodes (if applicable)


## Routing Strategy Overview

The NVIDIA LLM Router supports two routing strategies. For detailed architecture and component information, see the [main project documentation](../README.md#software-components).

### Manual Routing
- **Direct policy-based routing** to NVIDIA's hosted models via Build API
- **Requirements**: Standard OpenShift nodes, NVIDIA API key
- **Use case**: Production deployments, immediate functionality

### Triton Routing  
- **ML-based routing decisions** using local classification models
- **Requirements**: GPU nodes (16GB+ VRAM), NGC model downloads
- **Use case**: Custom routing logic, research environments

> **Recommendation**: Start with manual routing for immediate functionality. See [NVIDIA's routing strategy documentation](https://docs.nvidia.com/nim/llm-router/latest/routing.html) for detailed comparison.

## Deployment Steps

### 1. Create Required Secrets

```bash
# Create namespace (replace 'your-namespace' with your preferred name)
oc new-project your-namespace

# Set and validate environment variables
export NVIDIA_API_KEY="nvapi-YOUR-NVIDIA-API-KEY"
export NGC_API_KEY="YOUR-NGC-API-KEY"

# Validate NVIDIA API key format (should start with 'nvapi-' and be ~47 characters)
if [[ ! "$NVIDIA_API_KEY" =~ ^nvapi-.{40,}$ ]]; then
  echo "❌ Error: Invalid NVIDIA API key format. Should start with 'nvapi-' followed by 40+ characters"
  echo "Get your key from: https://build.nvidia.com/settings/api-keys"
  exit 1
fi

# Validate NGC API key format (should be alphanumeric, 40+ characters)
if [[ ! "$NGC_API_KEY" =~ ^[A-Za-z0-9]{40,}$ ]]; then
  echo "❌ Error: Invalid NGC API key format. Should be 40+ alphanumeric characters"
  echo "Get your key from: https://org.ngc.nvidia.com/setup/api-keys"
  exit 1
fi

echo "✅ API keys validated successfully"

# Create NVIDIA API key secret (required for all deployments)
oc create secret generic llm-api-keys \
  --from-literal=nvidia_api_key="$NVIDIA_API_KEY"

# Create NGC registry secret for pulling container images from nvcr.io
oc create secret docker-registry nvcr-secret \
  --docker-server=nvcr.io \
  --docker-username='$oauthtoken' \
  --docker-password="$NGC_API_KEY"

# Create NGC CLI secret for model downloads (required for Triton routing only)
oc create secret generic ngc-cli-key \
  --from-literal=ngc_cli_api_key="$NGC_API_KEY"
```

### 2. Choose Your Deployment Path

#### **Option A: Manual Routing (Recommended)**
**Cost-effective deployment with immediate functionality**

```bash
# Manual routing deployment (no GPU required)
helm install llm-router ./deploy/helm/llm-router \
  --set openshift.enabled=true \
  --set app.enabled=true \
  --set routerServer.enabled=false \
  --set routerController.enabled=true \
  --set global.imageRegistry=your-registry.com/namespace/ \
  --set routerController.image.tag=your-tag \
  --set app.image.tag=your-tag
```

> **Automatic OpenShift Configuration**: When `openshift.enabled=true`, the chart automatically configures:
> - **CephFS storage** (`ocs-storagecluster-cephfs`) with **ReadWriteMany** for concurrent pod access
> - **OpenShift Routes** instead of Kubernetes Ingress  
> - **Security contexts** compliant with `restricted-v2` SCC
> - **HuggingFace cache** redirection for proper permissions

#### **Option B: Triton Routing (Advanced)**
**GPU deployment with ML-based routing**

```bash
# Triton routing deployment (GPU required + cache configuration)
helm install llm-router ./deploy/helm/llm-router \
  --set openshift.enabled=true \
  --set app.enabled=true \
  --set routerServer.enabled=true \
  --set routerController.enabled=true \
  --set global.imageRegistry=your-registry.com/namespace/ \
  --set routerServer.image.tag=your-tag \
  --set routerController.image.tag=your-tag \
  --set app.image.tag=your-tag \
  --set routerServer.env[0].name=HF_DATASETS_CACHE \
  --set routerServer.env[0].value="/tmp/hf_cache" \
  --set routerServer.env[1].name=HUGGINGFACE_HUB_CACHE \
  --set routerServer.env[1].value="/tmp/hf_cache" \
  --set routerServer.env[2].name=TRANSFORMERS_CACHE \
  --set routerServer.env[2].value="/tmp/hf_cache" \
  --set routerServer.env[3].name=HOME \
  --set routerServer.env[3].value="/tmp"
```

> **Important Notes**:
> - **Manual routing** provides complete LLM routing functionality without GPU infrastructure
> - **Triton routing** requires GPU nodes and additional model setup (see section 4)
> - **Cache configuration** is only required for Triton routing (router-server component)
> - **Cost consideration**: Manual routing uses standard nodes, Triton routing requires expensive GPU nodes

### 3. Access the Application

```bash
# Get the route URLs
oc get routes

# Access the demo UI
https://llm-router-app-<your-namespace>.<cluster-domain>/
```

**Using the Application**:
- **Manual routing**: Works immediately - select "manual" routing strategy in the UI
- **Triton routing**: Requires additional model setup (see section 4) before using "triton" routing strategy

### 4. Triton Routing Model Setup (Advanced)

For environments that need Triton routing with ML-based routing decisions:

#### **Prerequisites for Model Downloads**

**Storage Requirements**:
- The deployment must use **ReadWriteMany (RWX) PVC** for model storage
- Required for shared access between download and inference pods
- **RHOAI Users**: If using RHOAI workbench, configure the LLM Router to use the same RWX PVC as your notebooks

**Verify PVC Configuration**:
```bash
# Verify the Helm-created PVC is RWX and bound
oc get pvc llm-router-router-server-model-repo -o custom-columns=NAME:.metadata.name,ACCESS:.spec.accessModes,STATUS:.status.phase --no-headers

# Expected output: llm-router-router-server-model-repo   [ReadWriteMany]   Bound
```

#### **Method 1: Direct Download Pod (Recommended)**

This method downloads NGC models directly to the PVC, avoiding file transfer issues:

**Step 1: Create Download ConfigMap**
```bash
# Create ConfigMap with Makefile for NGC downloads
# Note: Ensure you're in the project root directory containing the Makefile
oc create configmap download-makefile --from-file=Makefile -n your-namespace

# Verify Makefile was added correctly
oc describe configmap download-makefile | grep -A5 "Data"
```

**Step 2: Apply Download Pod**
```yaml
# download-models.yaml
apiVersion: v1
kind: Pod
metadata:
  name: ngc-model-downloader
  namespace: your-namespace
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

      # Copy models (702MB each)
      cp -v routers/task_router/1/model.pt /model_repository/task_router/1/model.pt
      cp -v routers/complexity_router/1/model.pt /model_repository/complexity_router/1/model.pt

      # Copy configuration and Python files with correct paths
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

      echo "✅ Models downloaded directly to PVC"
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
      claimName: llm-router-router-server-model-repo  # Created by the Helm chart (verify with: oc get pvc)
```

**Step 3: Run Download**
```bash
# Apply the download pod
oc apply -f download-models.yaml

# Monitor progress
oc logs -f ngc-model-downloader

# Wait for completion (pod runs to completion, not Ready)
oc wait --for=jsonpath='{.status.phase}'=Succeeded pod/ngc-model-downloader --timeout=600s

# IMPORTANT: Restart router-server to load the downloaded models
oc rollout restart deployment/llm-router-router-server

# Verify models loaded successfully
oc logs deployment/llm-router-router-server | grep "successfully loaded"
```

#### **Method 2: RHOAI Notebook Integration**

For users with RHOAI workbench environments:

1. **Configure Workbench**: Ensure your RHOAI workbench has the same RWX PVC attached that the LLM Router router-server uses for model storage

2. **Run Deployment Notebook**: Execute [launchable/1_Deploy_LLM_Router.ipynb](../launchable/1_Deploy_LLM_Router.ipynb) from your RHOAI workbench

3. **Verify Shared Storage**: The notebook will download models to the shared PVC, making them available to the LLM Router deployment

> **Note**: This method leverages RHOAI's notebook environment while ensuring models are downloaded to the shared storage accessible by the LLM Router pods.

> **Alternative**: Manual routing provides the same functionality without these file transfer complications.


## Architecture and Routing Strategies

The NVIDIA LLM Router supports two routing strategies. For detailed information about the architecture and components, see the main [README.md](../README.md#software-components).

### Key Differences for OpenShift Deployment

| Aspect | Manual Routing | Triton Routing |
|--------|----------------|----------------|
| **Components** | router-controller only | router-controller + router-server |
| **Infrastructure** | Standard nodes | GPU nodes (16GB+ VRAM) |
| **Setup Complexity** | Simple | Complex (model downloads) |
| **Cost** | Low | High (GPU instances) |
| **OpenShift Compatibility** | Excellent | Requires file transfer workarounds |

> **For complete architecture details and configuration options**, refer to:
> - [Main Documentation](../README.md) - Overall architecture and components
> - [Router Controller README](../src/router-controller/readme.md) - API and configuration details

## OpenShift-Specific Features

When `openshift.enabled: true`, the chart automatically configures:

### Security Context
- **Pod Level**: `runAsNonRoot: true`, `seccompProfile: RuntimeDefault`
- **Container Level**: `allowPrivilegeEscalation: false`, dropped capabilities
- **No hardcoded UIDs**: Lets OpenShift assign UIDs dynamically
- **SCC Compliance**: Works with `restricted-v2` SCC without cluster-admin privileges

### Networking
- **OpenShift Routes**: Created instead of standard Kubernetes Ingress
- **Edge TLS**: Automatic HTTPS termination
- **Path-based routing**: Multiple services accessible via different paths

### Storage
- **Default Storage Class**: Uses `gp3-csi` for OpenShift environments
- **PVC Creation**: Automatic persistent volume provisioning for model storage
- **Access Mode**: ReadWriteMany (RWX) required for Triton routing model downloads
- **RHOAI Integration**: Compatible with RHOAI workbench shared storage

### RBAC
- **ServiceAccount**: Dedicated service account with minimal permissions
- **Role/RoleBinding**: Basic permissions for ConfigMap and Secret access

## Configuration Options

### OpenShift Settings

```yaml
# values.yaml or --set overrides
openshift:
  enabled: true                    # Enable OpenShift-specific features
  storageClass: "ocs-storagecluster-cephfs"  # RWX storage class for model sharing
  routes:
    enabled: true                  # Create OpenShift Routes
    router:
      host: ""                    # Auto-generated if empty
      enabled: true
    controller:
      host: ""
      enabled: true
    app:
      host: ""
      enabled: true

# Router Server Storage (Required for Triton Routing)
routerServer:
  persistence:
    storageClass: "ocs-storagecluster-cephfs"  # Must be ReadWriteMany (RWX)
    size: "100Gi"                 # Sufficient for NGC models (~1.5GB total)
    accessMode: "ReadWriteMany"   # Required for model download pods
```

### Service Configuration

```yaml
# Enable/disable components
routerServer:
  enabled: true                    # Triton-based routing server (requires GPU)
  
routerController:
  enabled: true                    # API orchestration service
  
app:
  enabled: true                    # Optional Gradio demo UI
```

### Cache Configuration (Required for Triton Routing Only)

**Note**: This configuration is only needed when `routerServer.enabled=true` (Triton routing).

OpenShift security contexts prevent writing to the root filesystem, which causes HuggingFace cache permission errors in the router-server component. Configure proper cache directories:

```yaml
# values.yaml or --set overrides
routerServer:
  env:
    - name: HF_DATASETS_CACHE
      value: "/tmp/hf_cache"
    - name: HUGGINGFACE_HUB_CACHE
      value: "/tmp/hf_cache"
    - name: TRANSFORMERS_CACHE
      value: "/tmp/hf_cache"
    - name: HOME
      value: "/tmp"
  extraVolumes:
    - name: hf-cache
      emptyDir: {}
  extraVolumeMounts:
    - name: hf-cache
      mountPath: /tmp/hf_cache
```

**This cache configuration is already included in the Triton routing deployment command above.**

## Resource Requirements

### Manual Routing Deployment
| Component | CPU | Memory | GPU | Storage |
|-----------|-----|--------|-----|---------|
| router-controller | 0.5-1 core | 2Gi | - | - |
| app (demo) | 0.1-0.5 core | 1Gi | - | - |
| **Total** | **0.6-1.5 cores** | **3Gi** | **None** | **Standard PVC** |

### Triton Routing Deployment  
| Component | CPU | Memory | GPU | Storage |
|-----------|-----|--------|-----|---------|
| router-server | 2-4 cores | 8Gi | 1x NVIDIA GPU (16GB+ VRAM) | - |
| router-controller | 0.5-1 core | 2Gi | - | - |
| app (demo) | 0.1-0.5 core | 1Gi | - | - |
| model-storage | - | - | - | 100Gi PVC |
| **Total** | **2.6-5.5 cores** | **11Gi** | **1x GPU** | **100Gi+ PVC** |

> **Cost Impact**: GPU instances typically cost 10-50x more than standard instances in cloud environments.

## Troubleshooting

### Pod Security Issues
If pods fail with security context errors:

```bash
# Check SCC assignments
oc describe pod <pod-name>

# Verify service account
oc get serviceaccount llm-router
```

### Route Access Issues
If external access doesn't work:

```bash
# Check route status
oc get routes
oc describe route llm-router-app

# Test internal connectivity
oc port-forward service/llm-router-app 8008:8008
curl http://localhost:8008
```

### GPU Allocation Issues
If router-server pods are pending:

```bash
# Check GPU availability
oc describe node <gpu-node>

# Verify GPU operator
oc get pods -n nvidia-gpu-operator

# Check resource requests
oc describe pod <router-server-pod>
```

#### GPU Node Taints
Many OpenShift GPU nodes have taints that prevent non-GPU workloads from scheduling. If you see errors like:

```
6 node(s) had untolerated taint {g5-gpu: true}
```

Add the required toleration to your deployment:

```yaml
# gpu-tolerations.yaml
tolerations:
  - key: g5-gpu
    operator: Equal
    value: "true"
    effect: NoSchedule
```

Deploy Triton routing with tolerations:
```bash
# Option 1: Using a values file (recommended)
helm install llm-router ./deploy/helm/llm-router \
  --set openshift.enabled=true \
  --set app.enabled=true \
  --set routerServer.enabled=true \
  --set routerController.enabled=true \
  --set global.imageRegistry=your-registry.com/namespace/ \
  --set routerServer.image.tag=your-tag \
  --set routerController.image.tag=your-tag \
  --set app.image.tag=your-tag \
  --set 'routerServer.env[0].name=HF_DATASETS_CACHE' \
  --set 'routerServer.env[0].value=/tmp/hf_cache' \
  --set 'routerServer.env[1].name=HUGGINGFACE_HUB_CACHE' \
  --set 'routerServer.env[1].value=/tmp/hf_cache' \
  --set 'routerServer.env[2].name=TRANSFORMERS_CACHE' \
  --set 'routerServer.env[2].value=/tmp/hf_cache' \
  --set 'routerServer.env[3].name=HOME' \
  --set 'routerServer.env[3].value=/tmp' \
  -f gpu-tolerations.yaml

# Option 2: Using --set inline (note: use --set-string for the value field)
helm install llm-router ./deploy/helm/llm-router \
  --set openshift.enabled=true \
  --set app.enabled=true \
  --set routerServer.enabled=true \
  --set routerController.enabled=true \
  --set global.imageRegistry=your-registry.com/namespace/ \
  --set routerServer.image.tag=your-tag \
  --set routerController.image.tag=your-tag \
  --set app.image.tag=your-tag \
  --set 'routerServer.env[0].name=HF_DATASETS_CACHE' \
  --set 'routerServer.env[0].value=/tmp/hf_cache' \
  --set 'routerServer.env[1].name=HUGGINGFACE_HUB_CACHE' \
  --set 'routerServer.env[1].value=/tmp/hf_cache' \
  --set 'routerServer.env[2].name=TRANSFORMERS_CACHE' \
  --set 'routerServer.env[2].value=/tmp/hf_cache' \
  --set 'routerServer.env[3].name=HOME' \
  --set 'routerServer.env[3].value=/tmp' \
  --set 'tolerations[0].key=g5-gpu' \
  --set 'tolerations[0].operator=Equal' \
  --set-string 'tolerations[0].value=true' \
  --set 'tolerations[0].effect=NoSchedule'
```

> **Note**: Tolerations are applied at the top level (`.Values.tolerations`) and affect all components. When using `--set` inline, the `value` field must use `--set-string` to avoid Helm interpreting `true` as a boolean. Using `-f gpu-tolerations.yaml` avoids this issue.

> **Note**: GPU tolerations are only needed for Triton routing deployments.

Check node taints:
```bash
# List GPU nodes and their taints
oc get nodes -l nvidia.com/gpu.present=true -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints

# Check specific node
oc describe node <gpu-node-name> | grep Taints
```

### Model Repository Issues
If Triton routing fails to load models:

```bash
# Verify repository structure
oc exec deployment/llm-router-router-server -- find /model_repository -type f | head -20

# Check for actual model files (.pt files) - these are large (700MB+)
oc exec deployment/llm-router-router-server -- find /model_repository -name "*.pt" -exec ls -lh {} \;

# Check Triton server logs for detailed loading errors  
oc logs deployment/llm-router-router-server | grep -E "model|error|fail"
```

**Common Issues:**
1. **Missing model files** - Repository contains only template structure
2. **Corrupted transfers** - Large model files corrupted during pod-to-pod copy
3. **Cache permission errors** - HuggingFace models fail due to OpenShift security contexts
4. **Insufficient storage** - PVC too small for 700MB+ model files

**Solutions:**
- Ensure cache configuration is applied (see Cache Configuration section)
- Use temporary pod method for reliable large file transfers (see model download section)  
- Verify PVC has sufficient space (recommend 100Gi+)
- Consider manual routing as alternative

### Routing Strategy Issues

If routing doesn't work as expected:

#### **Manual Routing Issues**
```bash
# Check NVIDIA API key is set
oc get secret llm-api-keys -o yaml
oc logs deployment/llm-router-router-controller | grep -i "api"

# Test API connectivity from inside cluster
oc exec <router-controller-pod> -- curl -s https://api.nvidia.com/v1/models \
  -H "Authorization: Bearer ${NVIDIA_API_KEY}"

# Verify router controller is accessible
curl https://<router-controller-route>/v1/models
```

**Common Solutions:**
- Ensure NVIDIA API key is valid and has model access permissions
- Check network connectivity to api.nvidia.com
- Verify router-controller service is accessible via OpenShift route

#### **Triton Routing Issues**  
If using Triton routing and seeing "Request for unknown model" errors:

```bash
# Check if models are loaded in Triton
curl https://<router-server-route>/v2/models

# Check Triton server logs
oc logs deployment/llm-router-router-server | grep -E "model|loaded|failed"
```

**Common Solutions:**
- Verify NGC model files were transferred successfully (see model download section)
- Ensure cache configuration is applied (see Cache Configuration section)
- Check PVC has sufficient space for model files
- Use manual routing as alternative - provides same functionality

> **Recommendation**: For production deployments, manual routing provides reliable functionality without the complexity of model file management.

### Image Pull Issues
If images fail to pull:

```bash
# Verify NGC secret for base images
oc get secret nvcr-secret
oc describe secret nvcr-secret

# Check custom registry access
oc describe pod <pod-name>

# Verify image tags match your built images
helm template . --set global.imageRegistry=your-registry.com/
```

## GPU Configuration

### Node Selection and Tolerations

For clusters with GPU node taints, configure tolerations in your values:

```yaml
# values.yaml or separate tolerations file
tolerations:
  - key: g5-gpu                    # Common AWS g5 instance taint
    operator: Equal
    value: "true"
    effect: NoSchedule
  - key: nvidia.com/gpu           # Alternative GPU taint
    operator: Exists
    effect: NoSchedule

# Optional: Force scheduling only on GPU nodes
routerServer:
  nodeSelector:
    nvidia.com/gpu.present: "true"
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: nvidia.com/gpu.present
            operator: In
            values: ["true"]
```

### Resource Limits

Adjust GPU and memory requirements based on your models:

```yaml
# values.yaml
routerServer:
  resources:
    limits:
      nvidia.com/gpu: 1           # Number of GPUs required
      memory: "16Gi"              # Memory for large models
      cpu: "4"
    requests:
      nvidia.com/gpu: 1
      memory: "8Gi"
      cpu: "2"
```

## Image Building

For OpenShift deployment, you need to build container images:

### Images Required by Deployment Type

**Manual Routing**:
```bash
# Build router-controller image  
podman build --platform=linux/amd64 \
  -f src/router-controller/router-controller.dockerfile \
  -t your-registry.com/router-controller:your-tag .
podman push your-registry.com/router-controller:your-tag

# Build demo app image
podman build --platform=linux/amd64 \
  -f demo/app/app.dockerfile \
  -t your-registry.com/llm-router-client:your-tag .
podman push your-registry.com/llm-router-client:your-tag
```

**Triton Routing** (all manual routing images plus):
```bash
# Build router-server image (GPU support required)
podman build --platform=linux/amd64 \
  -f src/router-server/router-server.dockerfile \
  -t your-registry.com/router-server:your-tag .
podman push your-registry.com/router-server:your-tag
```

> **Architecture Note**: OpenShift typically runs on x86_64, so ensure images are built with `--platform=linux/amd64`.

## Advanced Configuration

### Custom Storage Classes

```yaml
# values.yaml
openshift:
  storageClass: "fast-ssd"        # Use custom storage class

# Or disable OpenShift auto-selection
routerServer:
  volumes:
    modelRepository:
      storage:
        persistentVolumeClaim:
          storageClass: "my-custom-class"
```

### Custom Hostnames

```yaml
# values.yaml
openshift:
  routes:
    app:
      host: "llm-demo.apps.your-cluster.com"
    controller:
      host: "llm-api.apps.your-cluster.com"
```

### Resource Limits

```yaml
# values.yaml
routerServer:
  resources:
    limits:
      nvidia.com/gpu: 1
      memory: "16Gi"
      cpu: "4"
    requests:
      nvidia.com/gpu: 1
      memory: "8Gi" 
      cpu: "2"
```

## Monitoring

### Health Checks
All components include health checks:

```bash
# Check component health
oc get pods
oc logs <pod-name>

# Test endpoints
curl https://<route-url>/health
```

### Resource Usage

```bash
# Monitor resource usage
oc top pods
oc top nodes

# GPU utilization (if available)
oc exec <router-server-pod> -- nvidia-smi
```

## Uninstall

```bash
# Remove Helm release
helm uninstall llm-router

# Clean up secrets (optional)
oc delete secret llm-api-keys nvcr-secret

# Clean up PVCs (optional - will delete model data)
oc delete pvc --selector=app.kubernetes.io/name=llm-router
```

## Support

For issues specific to:
- **OpenShift deployment**: Check this guide's troubleshooting section
- **NVIDIA GPU support**: Consult [NVIDIA GPU Operator documentation](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/)
- **LLM Router functionality**: See main project [README](../README.md)

## Version Compatibility

| LLM Router | OpenShift | NVIDIA GPU Operator | Notes |
|------------|-----------|-------------------|-------|
| 0.1.0+ | 4.10+ | 23.6.1+ | Initial OpenShift support |

Last updated: April 2026