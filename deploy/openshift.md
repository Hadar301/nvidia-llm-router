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
# Create namespace (replace 'your-namespace' with your preferred name)
oc new-project your-namespace

# Create NVIDIA API key secret (required for manual routing)
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
# Complete deployment (includes both routing strategies)
helm install llm-router ./deploy/helm/llm-router \
  --set openshift.enabled=true \
  --set app.enabled=true \
  --set routerServer.enabled=true \
  --set routerController.enabled=true \
  --set global.imageRegistry=your-registry.com/namespace/ \
  --set routerServer.image.tag=your-tag \
  --set routerController.image.tag=your-tag \
  --set app.image.tag=your-tag
```

> **Note**: 
> - Manual routing works immediately after deployment
> - GPU nodes with taints require additional tolerations (see [GPU Configuration](#gpu-configuration))  
> - Triton routing requires additional model download steps (see section 4)

### 3. Prepare for Model Routing

The NVIDIA LLM Router supports two routing approaches that work differently on OpenShift:

#### **Manual Routing (Recommended)**
- **Ready immediately**: Works with API keys, no model downloads required
- **Production ready**: Uses NVIDIA's Build API for reliable model access
- **OpenShift compatible**: No file transfer or storage issues

#### **Triton Routing (Advanced)**
- **Requires model files**: Needs NGC model downloads and file transfers
- **Complex setup**: File transfer challenges in OpenShift environments
- **Development focused**: Best suited for custom model development

For most users, **manual routing provides complete functionality** and is recommended for OpenShift deployments.

### 4. Download and Copy Models (Optional - For Triton Routing)

If you need Triton routing with NGC models, follow these steps:

#### **Step 1: Download Models Locally**

```bash
# Install NGC CLI and download models locally
export NGC_CLI_API_KEY="your-ngc-api-key"
export NGC_CLI_ORG="nvidia/nemo"

# Option A: Use Makefile (requires NGC CLI setup)
make download

# Option B: Use Jupyter notebook
jupyter lab --notebook-dir=launchable/
# Run 1_Deploy_LLM_Router.ipynb with NGC_API_Key set
```

#### **Step 2: Transfer Models to OpenShift**

⚠️ **Warning**: This process can be unreliable with large files. Model files are ~700MB each and may get corrupted during transfer.

```bash
# Get the router-server pod name
ROUTER_POD=$(oc get pods -l app.kubernetes.io/component=router-server -o jsonpath='{.items[0].metadata.name}')

# Copy model files directly to running pod - these operations may fail due to large file sizes
oc cp routers/task_router/1/model.pt $ROUTER_POD:/model_repository/task_router/1/
oc cp routers/complexity_router/1/model.pt $ROUTER_POD:/model_repository/complexity_router/1/

# Verify file sizes match (736MB each)
oc exec $ROUTER_POD -- ls -lh /model_repository/*/1/model.pt

# Restart router-server to reload models
oc rollout restart deployment/llm-router-router-server

# Verify models loaded successfully
oc logs deployment/llm-router-router-server | grep -E "successfully loaded"
```

**Common Issues**:
- `oc cp` may fail with "unexpected EOF" for large files
- File corruption during transfer (check file sizes)  
- Network timeouts with 700MB+ files
- May require multiple retry attempts

**Troubleshooting**:
```bash
# If transfer fails, verify source file size
ls -lh routers/*/1/model.pt

# Check if transferred file is complete (should be ~736MB each)
ROUTER_POD=$(oc get pods -l app.kubernetes.io/component=router-server -o jsonpath='{.items[0].metadata.name}')
oc exec $ROUTER_POD -- ls -lh /model_repository/*/1/model.pt

# If corrupted, delete and retry
oc exec $ROUTER_POD -- rm /model_repository/*/1/model.pt
```

> **Alternative**: Manual routing provides the same functionality without these file transfer complications.

### 5. Access the Application

```bash
# Get the route URLs
oc get routes

# Access the demo UI
https://llm-router-app-<your-namespace>.<cluster-domain>/
```

## Routing Strategies

The NVIDIA LLM Router supports two approaches for routing requests to language models:

### Manual Routing (Recommended)
**What it does**: Direct model selection based on predefined policies and user preferences

**Advantages**:
- ✅ **Ready immediately**: Works with just NVIDIA API keys
- ✅ **Reliable**: Uses NVIDIA's Build API infrastructure  
- ✅ **OpenShift compatible**: No file transfers or storage complexity
- ✅ **Production ready**: Proven reliability for enterprise deployments
- ✅ **Full functionality**: Complete LLM Router capabilities

**Use cases**:
- Production deployments
- Testing and evaluation
- When you want reliable model routing without infrastructure complexity

### Triton Routing (Advanced)
**What it does**: ML-based routing using Triton Inference Server with custom routing models

**Requirements**:
- ❗ **NGC model downloads**: Requires downloading ~700MB model files
- ❗ **File transfers**: Complex model file management in OpenShift
- ❗ **Storage overhead**: Significant PVC space requirements
- ❗ **Cache configuration**: HuggingFace model caching setup

**Use cases**:
- Custom model development
- Research environments
- When you need ML-based routing decisions

> **Recommendation**: Start with manual routing for immediate functionality. Triton routing adds operational complexity without additional end-user features for most use cases.

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
helm install llm-router ./deploy/helm/llm-router \
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
helm install llm-router ./deploy/helm/llm-router \
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