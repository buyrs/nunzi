# RunPod GPU Pods Runtime Integration - Design Document

## 1. Overview

This document outlines the design for integrating RunPod GPU Pods as a new runtime option in OpenHands. RunPod provides on-demand GPU compute (NVIDIA A100, H100, RTX GPUs) that can be dynamically provisioned and managed via their API.

## 2. Goals

- Enable OpenHands to provision GPU-enabled containers on RunPod
- Provide persistent compute for long-running agent tasks
- Support GPU-accelerated workloads (LLM inference, fine-tuning, etc.)
- Follow existing runtime patterns (similar to E2B, Modal, Daytona)

## 3. Architecture

### 3.1 High-Level Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                     OpenHands Server                            │
│                                                                 │
│  ┌──────────────┐    ┌─────────────────────────────────────┐  │
│  │ Agent Session│───▶│         RunpodRuntime               │  │
│  └──────────────┘    │                                     │  │
│                      │  - create_pod()                      │  │
│                      │  - connect()                         │  │
│                      │  - execute actions                   │  │
│                      │  - close()                           │  │
│                      └──────────────┬──────────────────────┘  │
│                                     │                          │
│                                     ▼                          │
│                      ┌─────────────────────────────────────┐  │
│                      │         RunPod API                   │  │
│                      │   (create/manage Pods)              │  │
│                      └──────────────┬──────────────────────┘  │
│                                     │                          │
│                                     ▼                          │
│                      ┌─────────────────────────────────────┐  │
│                      │      GPU Pod (Container)             │  │
│                      │  - Action Execution Server           │  │
│                      │  - Python/Bash environment           │  │
│                      │  - GPU access (optional)             │  │
│                      └─────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Component Structure

```
third_party/runtime/impl/runpod/
├── __init__.py              # Runtime registration
├── runpod_runtime.py        # Main RunpodRuntime class
├── pod_manager.py           # RunPod API interactions
├── filestore.py            # File operations (SSH/SFTP)
└── README.md               # Documentation
```

## 4. Configuration

### 4.1 Environment Variables

| Variable | Required | Description | Default |
|----------|----------|-------------|---------|
| `RUNPOD_API_KEY` | Yes | RunPod API key | - |
| `RUNPOD_TEMPLATE_ID` | No | Pre-configured template ID | - |
| `RUNPOD_IMAGE_NAME` | No | Docker image (e.g., `runpod/pytorch:2.1`) | `runpod/base:0.5.0` |
| `RUNPOD_GPU_TYPE` | No | GPU type (e.g., `NVIDIA A100 80GB PCIe`) | `NVIDIA RTX A6000` |
| `RUNPOD_GPU_COUNT` | No | Number of GPUs | 1 |
| `RUNPOD_VOLUME_IN_GB` | No | Persistent volume size | 0 |
| `RUNPOD_NETWORK_VOLUME_ID` | No | Network volume ID for persistent storage | - |
| `RUNPOD_CONTAINER_DISK_IN_GB` | No | Container disk size | 50 |
| `RUNPOD_LOCATION` | No | Data center location | - |
| `RUNPOD_SECURE_CLOUD` | No | Use secure cloud (vs community) | `false` |
| `RUNPOD_IDLE_TIMEOUT` | No | Minutes before auto-stop | 30 |
| `RUNPOD_SSH_KEY` | No | SSH public key for access | - |

### 4.2 Sandbox Config Extension

```python
# openhands/core/config/sandbox_config.py additions
class SandboxConfig:
    # ... existing fields ...
    
    # RunPod-specific configuration
    runpod_api_key: str | None = None
    runpod_template_id: str | None = None
    runpod_image_name: str | None = None
    runpod_gpu_type: str | None = None
    runpod_gpu_count: int = 1
    runpod_volume_in_gb: int = 0
    runpod_network_volume_id: str | None = None
    runpod_container_disk_in_gb: int = 50
    runpod_location: str | None = None
    runpod_secure_cloud: bool = False
    runpod_idle_timeout: int = 30
```

## 5. Runtime Implementation

### 5.1 Class Definition

```python
class RunpodRuntime(ActionExecutionClient):
    """Runtime that provisions GPU pods on RunPod."""
    
    def __init__(
        self,
        config: OpenHandsConfig,
        event_stream: EventStream,
        llm_registry: LLMRegistry,
        sid: str = "default",
        plugins: list[PluginRequirement] | None = None,
        env_vars: dict[str, str] | None = None,
        status_callback: Callable | None = None,
        attach_to_existing: bool = False,
        headless_mode: bool = True,
        user_id: str | None = None,
        git_provider_tokens: PROVIDER_TOKEN_TYPE | None = None,
    ):
        # Initialize parent class
        super().__init__(...)
        
        # RunPod-specific
        self.pod = None
        self.pod_id = None
        self.ssh_client = None
        self.sftp_client = None
        self._runtime_initialized = False
```

### 5.2 Key Methods

#### connect()
1. Initialize RunPod API client
2. Create or attach to existing Pod:
   - If `attach_to_existing=True` and `pod_id` cached → connect to existing
   - Otherwise → create new Pod with configured specs
3. Wait for Pod to be ready (poll status)
4. Establish SSH connection to Pod
5. Start action execution server inside Pod (if not already running)
6. Configure workspace mount path
7. Set runtime status to READY

#### run(action: CmdRunAction) → Observation
- Execute command via SSH
- Return CmdOutputObservation with stdout/stderr/exit code

#### run_ipython(action: IPythonRunCellAction) → Observation
- Execute Python code via SSH or direct Python interpreter
- Return IPythonRunCellObservation

#### read(action: FileReadAction) → Observation
- Use SFTP to read file from Pod
- Return FileReadObservation

#### write(action: FileWriteAction) → Observation
- Use SFTP to write file to Pod
- Return FileWriteObservation

#### browse(action: BrowseURLAction) → Observation
- Use curl/wget via SSH to fetch URL
- Return BrowserOutputObservation

#### close()
1. Optionally stop/delete Pod (configurable)
2. Close SSH/SFTP connections
3. Update cache

### 5.3 Pod Lifecycle Management

```python
class PodManager:
    """Manages RunPod pod lifecycle."""
    
    def __init__(self, api_key: str):
        self.client = runpod.Client(api_key)
    
    def create_pod(
        self,
        image_name: str,
        gpu_type: str,
        gpu_count: int = 1,
        volume_in_gb: int = 0,
        container_disk_in_gb: int = 50,
        location: str | None = None,
        secure_cloud: bool = False,
        template_id: str | None = None,
        env: dict[str, str] | None = None,
    ) -> dict:
        """Create a new GPU pod."""
        # Use template or custom image
        if template_id:
            return self.client.create_pod_from_template(...)
        else:
            return self.client.create_pod(
                name=f"openhands-{uuid.uuid4().hex[:8]}",
                image_name=image_name,
                gpu_type_id=gpu_type,
                gpu_count=gpu_count,
                volume_in_gb=volume_in_gb,
                container_disk_in_gb=container_disk_in_gb,
                location=location,
                secure_cloud=secure_cloud,
                env=env,
            )
    
    def get_pod(self, pod_id: str) -> dict:
        """Get pod status and details."""
        return self.client.get_pod(pod_id)
    
    def wait_until_ready(self, pod_id: str, timeout: int = 300) -> bool:
        """Poll until pod is running."""
        while timeout > 0:
            pod = self.get_pod(pod_id)
            if pod.get('status') == 'RUNNING':
                return True
            time.sleep(5)
            timeout -= 5
        return False
    
    def stop_pod(self, pod_id: str) -> None:
        """Stop a running pod."""
        self.client.stop_pod(pod_id)
    
    def delete_pod(self, pod_id: str) -> None:
        """Delete a pod."""
        self.client.delete_pod(pod_id)
```

## 6. Storage Options

### 6.1 Volume Disk (Default)
- Mounted at `/workspace` by default
- Persists when Pod is stopped
- Data deleted when Pod is deleted

### 6.2 Network Volume (Optional)
- Use `RUNPOD_NETWORK_VOLUME_ID` to attach
- Persists independently of Pod lifecycle
- Can be shared across multiple Pods
- Recommended for persistent projects

### 6.3 File Transfer
- Use SFTP for file operations
- Implement `RunpodFileStore` class wrapping `pysftp`

## 7. Network & Connectivity

### 7.1 SSH Access
- Connect to Pod via SSH using RunPod's proxy
- Host: `{pod_id}.proxy.runpod.net`
- Port: 22 (default)
- Authentication: API key or configured SSH key

### 7.2 Port Exposure
- RunPod automatically proxies exposed ports
- Access: `https://{pod-id}-{port}.proxy.runpod.net`
- Action execution server runs on configurable port (default: 8000)

### 7.3 Action Execution Server
- Must be pre-installed in container image OR
- Installed via startup script on first run
- Communicate via HTTP over SSH tunnel or direct pod network

## 8. GPU Support

### 8.1 GPU Types
Support all RunPod GPU types:
- NVIDIA A100 80GB (PCIe & SXM)
- NVIDIA H100 80GB
- NVIDIA RTX A6000
- NVIDIA RTX 4090
- And more...

### 8.2 GPU Detection
```python
def detect_gpu() -> str | None:
    """Detect available GPU in container."""
    result = subprocess.run(
        ['nvidia-smi', '--query-gpu=name', '--format=csv,noheader'],
        capture_output=True
    )
    if result.returncode == 0:
        return result.stdout.decode().strip()
    return None
```

### 8.3 CUDA Configuration
- Use RunPod's pre-built CUDA images
- Or specify custom image with required CUDA version

## 9. Error Handling

| Error | Handling |
|-------|----------|
| Pod creation failed | Retry with exponential backoff, raise RuntimeError |
| Pod timeout | Delete failed pod, raise TimeoutError |
| SSH connection failed | Retry connection 3 times, fallback to HTTP |
| Command execution failed | Return ErrorObservation with details |
| GPU not available | Fallback to CPU-only or raise error |

## 10. Caching & Reuse

### 10.1 Pod ID Cache
- Cache pod ID per session (`sid`)
- Reuse existing pod if `attach_to_existing=True`
- Benefits: Faster startup, cost savings

### 10.2 Template Caching
- Allow users to pre-create templates
- Skip image build on subsequent runs

## 11. Security Considerations

- API key stored securely (env var, not logged)
- SSH key-based authentication preferred
- Network isolation via RunPod's secure cloud option
- No sensitive data in pod logs

## 12. Cost Optimization

| Strategy | Description |
|----------|-------------|
| Idle timeout | Auto-stop after N minutes of inactivity |
| Spot instances | Use community cloud for lower rates |
| Volume cleanup | Delete volumes when no longer needed |
| Caching | Reuse pods across sessions |

## 13. Monitoring & Observability

### 13.1 Logging
- Pod creation/deletion events
- Connection status changes
- Command execution (debug level)
- Errors and failures

### 13.2 Metrics (Future)
- Pod runtime duration
- Command execution time
- Cost per session
- GPU utilization

## 14. Implementation Phases

### Phase 1: Core Functionality
- [ ] Create RunpodRuntime class
- [ ] Implement pod creation/deletion
- [ ] SSH connection handling
- [ ] Basic command execution
- [ ] File read/write operations

### Phase 2: Enhanced Features
- [ ] SFTP file operations
- [ ] Browse/interactive actions
- [ ] IPython execution
- [ ] Pod reuse/caching

### Phase 3: Advanced Features
- [ ] GPU support and detection
- [ ] Network volume support
- [ ] Cost tracking
- [ ] Metrics/observability

### Phase 4: Polish
- [ ] Error handling improvements
- [ ] Documentation
- [ ] Tests
- [ ] CI/CD integration

## 15. Dependencies

```toml
# pyproject.toml additions
[project.dependencies]
runpod = "^1.7.0"       # RunPod SDK
pysftp = "^0.2.9"       # SFTP operations
paramiko = "^3.0.0"     # SSH client (if needed)
```

## 16. Alternative Approaches

### Option A: SSH-based (Recommended)
- Connect via SSH to running pod
- Execute commands over SSH
- Use SFTP for file transfer
- Pros: Simple, reliable, well-tested
- Cons: SSH overhead

### Option B: HTTP-based
- Expose action server on pod port
- Connect via HTTP through RunPod proxy
- Pros: Lower latency
- Cons: Requires port exposure, less secure

### Option C: RunPod Serverless
- Use RunPod serverless endpoints
- Pros: Pay-per-request, auto-scaling
- Cons: Not suitable for persistent sessions
- Use case: Short-lived tasks only

## 17. Open Questions

1. **Should pods persist between sessions?**
   - Current design: Pods can be reused via `attach_to_existing`
   - Alternative: Always create fresh pod

2. **How to handle action execution server?**
   - Option A: Pre-install in container image
   - Option B: Install via startup script
   - Option C: Use existing server if available

3. **What container image to use?**
   - Use RunPod base image and install dependencies
   - Or provide custom OpenHands-specific image

4. **Idle timeout behavior?**
   - Stop pod after N minutes of inactivity?
   - Or require explicit close()?

5. **Cost limits?**
   - Should we implement max cost per session?
   - Budget alerts?
