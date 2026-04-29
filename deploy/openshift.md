# NVIDIA LLM Router - OpenShift Deployment Guide

This guide explains how to deploy the NVIDIA LLM Router on OpenShift using the provided Helm chart with OpenShift-specific configurations.

## Prerequisites

- OpenShift 4.10+ cluster with GPU nodes
- Helm 3.2.0+
- NVIDIA GPU Operator installed on OpenShift
- [NVIDIA API key](https://build.nvidia.com/settings/api-keys) from NVIDIA API Catalog
- [NGC API key](https://org.ngc.nvidia.com/setup/api-keys) for container registry access (base images)
- Container images for router-server, router-controller, and app components
- GPU node tolerations configured (if using tainted GPU nodes)

## Quick Start

### 1. Create Required Secrets

```bash
# Create NVIDIA API key secret
oc create secret generic llm-api-keys \
  --from-literal=nvidia_api_key=nvapi-YOUR-NVIDIA-API-KEY

# Create NGC registry secret for NVIDIA base images
oc create secret docker-registry nvcr-secret \
  --docker-server=nvcr.io \
  --docker-username='$oauthtoken' \
  --docker-password=YOUR-NGC-API-KEY
```

### 2. Deploy with OpenShift Support

```bash
# Deploy with OpenShift enablement and custom images
helm install llm-router ./helm/llm-router \
  --set openshift.enabled=true \
  --set app.enabled=true \
  --set routerServer.enabled=true \
  --set routerController.enabled=true \
  --set global.imageRegistry=your-registry.com/namespace/ \
  --set routerServer.image.tag=your-tag \
  --set routerController.image.tag=your-tag \
  --set app.image.tag=your-tag \
  --set routerServer.replicas=1
```

> **Note**: If your GPU nodes have taints (common with g5 instances), you'll need to add tolerations. See the [GPU Configuration](#gpu-configuration) section below.

### 3. Initialize Model Repository

The LLM Router requires **both template structure and actual trained models** in the model repository. The repository provides templates but you must download the actual models separately.

#### **Option A: Full Setup (Templates + NGC Models)**

For complete Triton routing functionality, use the Jupyter notebook approach:

```bash
# 1. Set up NGC API key and run the blueprint notebook locally
export NGC_CLI_API_KEY="your-ngc-api-key"
jupyter lab --notebook-dir=launchable/

# 2. Run 1_Deploy_LLM_Router.ipynb cells to download models
# 3. Copy downloaded models to OpenShift PVC (see below)
```

#### **Option B: Copy Local Templates (Manual Routing)**

For immediate testing with manual routing only, copy templates from the local repository:

```bash
# Scale down router-server to access PVC
oc scale deployment llm-router-router-server --replicas=0 -n your-namespace

# Copy model templates from local repository
oc cp customize/router-builder/task-router/triton_template/ <temp-pod>:/model_repository/
oc cp customize/router-builder/complexity-router/triton_template/ <temp-pod>:/model_repository/

# Scale router-server back up
oc scale deployment llm-router-router-server --replicas=1 -n your-namespace

echo "✅ Model templates copied"
echo "⚠️  Note: Templates only - use manual routing strategy for testing"
```

#### **Copying Downloaded Models to OpenShift**

After running the notebook locally to download NGC models:

```bash
# 1. Scale down router-server to access PVC
oc scale deployment llm-router-router-server --replicas=0 -n your-namespace

# 2. Copy downloaded models from local routers/ directory
oc cp routers/task_router/1/model.pt <router-server-pod>:/model_repository/task_router/1/
oc cp routers/complexity_router/1/model.pt <router-server-pod>:/model_repository/complexity_router/1/

# 3. Scale router-server back up
oc scale deployment llm-router-router-server --replicas=1 -n your-namespace

# 4. Verify models loaded successfully
oc logs deployment/llm-router-router-server | grep -E "model.*loaded|ready"
```

### 4. Access the Application

```bash
# Get the route URLs
oc get routes

# Access the demo UI
https://llm-router-app-<your-namespace>.<cluster-domain>/
```

## Routing Strategies

The NVIDIA LLM Router supports two routing strategies:

### Manual Routing (Recommended for Production)
- **Direct model selection**: Choose specific models from the configured policies
- **Uses NVIDIA API directly**: Routes to models via NVIDIA's Build API
- **Immediate functionality**: Works out-of-the-box with proper API keys
- **Best for**: Production deployments, specific model requirements

### Triton Routing (Requires Trained Models)  
- **ML-based routing**: Uses Triton Inference Server with routing models
- **Requires training**: Needs actual trained routing models (not provided)
- **Complex setup**: Requires model training and Triton configuration
- **Best for**: Advanced use cases with custom-trained routing logic

> **Default Configuration**: The demo app uses manual routing by default since the Triton routing models in the repository are template/stub models for development purposes only.

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

### RBAC
- **ServiceAccount**: Dedicated service account with minimal permissions
- **Role/RoleBinding**: Basic permissions for ConfigMap and Secret access

## Configuration Options

### OpenShift Settings

```yaml
# values.yaml or --set overrides
openshift:
  enabled: true                    # Enable OpenShift-specific features
  storageClass: "gp3-csi"         # Default OpenShift storage class
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

### Cache Configuration (Required for OpenShift)

OpenShift security contexts prevent writing to the root filesystem, which causes HuggingFace cache permission errors. Configure proper cache directories:

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

**Deploy with cache configuration:**
```bash
helm install llm-router ./helm/llm-router \
  --set openshift.enabled=true \
  --set routerServer.env[0].name=HF_DATASETS_CACHE \
  --set routerServer.env[0].value="/tmp/hf_cache" \
  --set routerServer.env[1].name=HUGGINGFACE_HUB_CACHE \
  --set routerServer.env[1].value="/tmp/hf_cache" \
  --set routerServer.env[2].name=TRANSFORMERS_CACHE \
  --set routerServer.env[2].value="/tmp/hf_cache" \
  --set routerServer.env[3].name=HOME \
  --set routerServer.env[3].value="/tmp"
```

## Resource Requirements

| Component | CPU | Memory | GPU | Storage |
|-----------|-----|--------|-----|---------|
| router-server | 2-4 cores | 8Gi | 1x NVIDIA GPU (16GB+ VRAM) | - |
| router-controller | 0.5-1 core | 2Gi | - | - |
| app (demo) | 0.1-0.5 core | 1Gi | - | - |
| model-storage | - | - | - | 100Gi PVC |

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

Deploy with tolerations:
```bash
helm install llm-router ./helm/llm-router \
  --set openshift.enabled=true \
  --set routerServer.enabled=true \
  --set global.imageRegistry=your-registry.com/namespace/ \
  --set routerServer.image.tag=your-tag \
  --set routerController.image.tag=your-tag \
  --set app.image.tag=your-tag \
  -f gpu-tolerations.yaml
```

Check node taints:
```bash
# List GPU nodes and their taints
oc get nodes -l nvidia.com/gpu.present=true -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints

# Check specific node
oc describe node <gpu-node-name> | grep Taints
```

### Model Repository Issues
If Triton fails to load models:

```bash
# Check if template loader job completed successfully
oc get jobs
oc logs job/llm-router-template-loader

# Verify repository structure
oc exec deployment/llm-router-router-server -- find /model_repository -type f | head -20

# Check for actual model files (.pt files)
oc exec deployment/llm-router-router-server -- find /model_repository -name "*.pt"

# Check Triton server logs for detailed loading errors  
oc logs deployment/llm-router-router-server | grep -E "model|error|fail"

# Common issues:
# 1. Empty .gitkeep files instead of real models
# 2. Templates present but no NGC model files
# 3. Cache permission errors for preprocessing models
```

### Triton Routing Not Working

If you see "Request for unknown model" errors, this typically means Triton has no loaded models:

#### **Root Cause: Missing Model Files**

The repository contains **template configurations** but requires downloading **actual trained models** from NGC:

```bash
# Check if models are loaded in Triton
curl https://<router-server-route>/v2/models

# Check Triton logs for loading errors  
oc logs deployment/llm-router-router-server
```

Common error messages and solutions:

**1. "failed to load all models"** - No valid models in repository:
- Repository only contains template structure
- Need to download actual `.pt` model files from NGC

**2. "PytorchStreamReader failed reading zip archive: not a ZIP archive"** - Invalid model files:
- Placeholder or corrupted model files
- Need actual PyTorch model files from `make download`

**3. "PermissionError: [Errno 13] Permission denied: '/.cache'"** - Cache permissions:
- **Root Cause**: OpenShift security contexts prevent writing to root filesystem (`/.cache`)
- **Impact**: HuggingFace Python preprocessing/postprocessing models fail to download
- **Symptom**: Triton ensemble models fail to load, preventing ML-based routing
- **Solution**: Deploy with cache configuration (see [Cache Configuration](#cache-configuration-required-for-openshift) section)
- This is **required for all OpenShift deployments** due to security context restrictions

#### **Solution: Download NGC Models**

The blueprint requires running `make download` to populate actual models:

```bash
# Download models locally (requires NGC CLI and API key)
export NGC_CLI_API_KEY="your-ngc-key"
export NGC_CLI_ORG="nvidia/nemo"
make download

# Or use Jupyter notebook approach:
# Run launchable/1_Deploy_LLM_Router.ipynb with NGC_API_Key set
```

#### **Alternative: Use Manual Routing**

Manual routing works immediately without NGC models:

1. **Switch to manual routing strategy in the Gradio UI**
2. **Test with OpenAI-compatible API calls directly**
3. **Verify router-controller configuration has valid NVIDIA_API_KEY**

Manual routing provides **complete LLM Router functionality** for evaluation and testing.

### Understanding Triton Inference Server Failures in OpenShift

This section explains the specific challenges encountered when deploying Triton Inference Server on OpenShift and how to resolve them.

#### **The Complete Failure Scenario**

When deploying the NVIDIA LLM Router on OpenShift, you may encounter this sequence of failures:

1. **Triton starts successfully** but shows "failed to load all models"
2. **Models table is empty** when checking `/v2/models` endpoint
3. **ML-based routing fails** with "Request for unknown model" errors
4. **Only manual routing works** via router-controller

#### **Root Cause Analysis**

The failure occurs due to **OpenShift's security restrictions** interacting with **HuggingFace model downloads**:

```bash
# Typical error in Triton logs:
E0428 12:20:22.188450 1 model_repository_manager.cc:1460] 
"Poll failed for model directory 'lost+found': Invalid model name"

# Hidden underlying cause (not visible in Triton logs):
PermissionError: [Errno 13] Permission denied: '/.cache'
```

**Technical Details:**

1. **OpenShift Security Contexts**: 
   - `runAsNonRoot: true` prevents root filesystem writes
   - `allowPrivilegeEscalation: false` restricts privilege changes
   - `restricted-v2` SCC blocks access to `/` directory

2. **HuggingFace Behavior**:
   - Python preprocessing models attempt to cache to `~/.cache/huggingface/`
   - When `HOME=/` (default), this becomes `/.cache/huggingface/`
   - OpenShift blocks this write operation

3. **Triton Ensemble Impact**:
   - Ensemble models depend on preprocessing/postprocessing steps
   - Cache permission errors cause Python backend models to fail
   - Failed dependencies prevent entire ensemble from loading

#### **The Solution: Cache Redirection**

The fix involves redirecting HuggingFace cache to a writable location:

**Environment Variables:**
```yaml
env:
  - name: HF_DATASETS_CACHE
    value: "/tmp/hf_cache"
  - name: HUGGINGFACE_HUB_CACHE
    value: "/tmp/hf_cache"
  - name: TRANSFORMERS_CACHE
    value: "/tmp/hf_cache"
  - name: HOME
    value: "/tmp"
```

**Volume Configuration:**
```yaml
extraVolumes:
  - name: hf-cache
    emptyDir: {}
extraVolumeMounts:
  - name: hf-cache
    mountPath: /tmp/hf_cache
```

#### **Verification Steps**

After applying the cache fix, verify it's working:

```bash
# 1. Check environment variables are set
oc exec <router-server-pod> -- env | grep -E "(HF_|TRANSFORM|HOME)"

# Expected output:
# HF_DATASETS_CACHE=/tmp/hf_cache
# HUGGINGFACE_HUB_CACHE=/tmp/hf_cache  
# TRANSFORMERS_CACHE=/tmp/hf_cache
# HOME=/tmp

# 2. Check cache directory is writable
oc exec <router-server-pod> -- ls -la /tmp/hf_cache

# Expected output:
# drwxrwsrwx. 2 root 1000970000  6 Apr 28 12:20 .

# 3. Check Triton can now load models (after adding model files)
curl https://<router-server-route>/v2/models
```

#### **Why This Wasn't Obvious**

This issue was particularly challenging to diagnose because:

1. **Triton logs don't show the real error** - they show model loading failures, not permission errors
2. **The error occurs in Python backends** - hidden from main Triton process logs  
3. **Manual routing still works** - making it seem like a Triton-specific issue
4. **Standard Kubernetes deployments work** - the issue is OpenShift-specific

#### **Implementation in Helm Chart**

The cache configuration is now built into the helm chart (not a post-deployment patch):

```bash
# Deploy with cache configuration from start:
helm install llm-router ./helm/llm-router \
  --set openshift.enabled=true \
  --set routerServer.env[0].name=HF_DATASETS_CACHE \
  --set routerServer.env[0].value="/tmp/hf_cache" \
  # ... additional cache environment variables
```

Or use a values file:
```yaml
# cache-config.yaml
routerServer:
  env:
    - name: HF_DATASETS_CACHE
      value: "/tmp/hf_cache"
    # ... other cache variables
  extraVolumes:
    - name: hf-cache
      emptyDir: {}
  extraVolumeMounts:
    - name: hf-cache
      mountPath: /tmp/hf_cache
```

This cache fix is **mandatory for all OpenShift deployments** and should be included from the initial installation, not applied as an afterthought.

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

For OpenShift deployment, you need to build multi-architecture container images:

### Build Router Components

```bash
# Build router-server image
podman build --platform=linux/amd64 \
  -f src/router-server/router-server.dockerfile \
  -t your-registry.com/router-server:your-tag .
podman push your-registry.com/router-server:your-tag

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