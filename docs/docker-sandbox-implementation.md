# Docker Sandbox Implementation

YokeFlow uses Docker containers to isolate agent execution and prevent environment variable leakage.

---

## Quick Reference

**Current Implementation:**
- ✅ Container reuse between coding sessions (instant startup)
- ✅ Clean slate for initializer sessions (full recreation)
- ✅ Volume mounts for file persistence
- ✅ Auto-stop on project completion (December 2025)
- ✅ Auto-delete on project deletion (December 2025)
- ✅ Web UI container management (/containers page)

**Container Lifecycle:**
```
Initializer → Creates new container → Kept running
Coding → Reuses container → Kept running
Coding → Reuses container → Kept running
Project Complete → ✅ Container auto-stopped (frees ports)
Project Deleted → ✅ Container auto-deleted (cleanup)
```

---

## Container Lifecycle

### Current Implementation (Container Reuse)

**✅ IMPLEMENTED:** Containers are reused between sessions for speed.

**Initializer Sessions:**
1. Check for existing container with name `yokeflow-{project}`
2. If exists → Remove and recreate (clean slate)
3. Create new container with volume mount
4. Run setup (install git, curl, build-essential, etc.)
5. Keep running after session ends

**Coding Sessions:**
1. Check for existing container with name `yokeflow-{project}`
2. If running → Reuse it (clean up processes only)
3. If stopped → Restart it
4. If missing → Create new one
5. Keep running after session ends

**Benefits:**
- Coding sessions start instantly (<5s vs 30-60s)
- Packages, node_modules preserved between sessions
- No repeated apt-get/npm installs

**Code Reference:** [sandbox_manager.py:200-258](../sandbox_manager.py#L200-L258)

### Container Management (December 2025)

**✅ IMPLEMENTED:** Automatic cleanup and manual controls.

**Auto-Stop on Completion:**
- When all tasks complete, container automatically stopped
- Frees ports (3000, 5000, etc.) for other projects
- Preserves environment for potential restart
- Implemented in `orchestrator.py` (lines 484-498)

**Auto-Delete on Project Deletion:**
- When project deleted via API, container automatically removed
- Frees disk space and cleans up Docker Desktop
- Implemented in `SandboxManager.delete_docker_container()`

**Manual Controls** (/containers page):
- View all Docker containers across all projects
- Start/Stop/Delete controls
- Real-time status display
- Port mappings display
- Statistics dashboard

**Benefits:**
- No port conflicts between projects
- Reduced resource usage
- Clean Docker Desktop interface
- No manual cleanup needed

### Manual Cleanup

**View all containers:**
```bash
docker ps -a --filter "name=yokeflow"
```

**Remove specific container:**
```bash
docker rm -f yokeflow-{project-name}
```

**Remove all stopped containers:**
```bash
docker container prune -f
```

**Remove all yokeflow containers:**
```bash
docker ps -a --filter "name=yokeflow" --format "{{.Names}}" | xargs docker rm -f
```

---

## Why Docker?

### Problem: Environment Variable Leakage

When agent generates apps with API keys, those environment variables leak into the parent process:

```
Session 1: Agent generates chatbot with ANTHROPIC_API_KEY
Session 2: Agent tests chatbot by running it
Session 3: ANTHROPIC_API_KEY from chatbot now in agent's environment
Session 4: Agent fails with "credit balance too low" (using chatbot's key)
```

### Solution: Docker Isolation

- **Separate filesystem** - no path conflicts
- **Separate environment** - no variable leakage
- **Volume mounts** - file persistence across sessions
- **No session limits** - containers run indefinitely
- **Zero cloud costs** - runs on local hardware

### vs. Cloud Alternatives

| Feature | Docker (Local) | E2B Cloud |
|---------|---------------|-----------|
| Session duration | Unlimited | 1hr (free), 24hr (pro) |
| Cost | $0 | $0.0001/second |
| Startup time | <5s (reuse) | ~150ms |
| File sync | Instant (volume mount) | Upload/download |
| Network latency | None | API calls |
| Control | Full (docker CLI) | API only |

**Winner:** Docker for long-running autonomous coding sessions.

---

## How It Works

### 1. Container Creation

**File:** `sandbox_manager.py` - `DockerSandbox.start()`

```python
# Generate unique name per project
container_name = f"yokeflow-{project_dir.name}"

# Check for existing container
try:
    existing = client.containers.get(container_name)

    # Initializer: Always recreate (clean slate)
    if session_type == "initializer":
        existing.remove(force=True)
        existing = None

    # Coding: Reuse if possible (speed)
    elif session_type == "coding":
        if existing.status == "running":
            # Clean up processes, reuse container
            await _cleanup_container()
            return  # Skip creation
        else:
            existing.start()  # Restart stopped container
            return

except docker.errors.NotFound:
    pass  # Create new container below

# Create new container
container = client.containers.run(
    image="node:20-slim",
    command="sleep infinity",  # Keep alive
    name=container_name,
    detach=True,
    volumes={
        str(project_dir): {
            "bind": "/workspace",
            "mode": "rw"
        }
    },
    working_dir="/workspace",
    mem_limit="2g",
    nano_cpus=2_000_000_000,  # 2.0 CPU
)
```

### 2. Process Cleanup (Coding Sessions)

When reusing containers, clean up stale processes:

```python
async def _cleanup_container(self):
    """Kill orphaned processes from previous sessions."""
    cleanup_commands = [
        "pkill -9 node || true",
        "pkill -9 npm || true",
        "pkill -9 python || true",
        "lsof -ti:3001 | xargs kill -9 || true",
        "lsof -ti:5173 | xargs kill -9 || true",
    ]

    for cmd in cleanup_commands:
        await self.execute_command(cmd)
```

**Why this matters:**
- Previous session may have left dev servers running
- Ports 3001, 5173 may be occupied
- Kill them before starting new session

### 3. MCP bash_docker Tool

Agent uses `bash_docker` tool instead of regular `Bash`:

```typescript
// mcp-task-manager/src/index.ts
case 'bash_docker':
    const containerName = process.env.DOCKER_CONTAINER_NAME;
    const command = args?.command as string;

    // Execute via docker exec
    const dockerCommand = `docker exec ${containerName} /bin/bash -c ${JSON.stringify(command)}`;
    const { stdout, stderr } = await execAsync(dockerCommand);

    return {
        content: [{ type: 'text', text: stdout + stderr }]
    };
```

**How agent learns to use it:**
1. System prompt includes Docker guidance when container active
2. MCP server registers `bash_docker` tool
3. Agent sees tool and chooses it automatically

**Code References:**
- Tool implementation: [mcp-task-manager/src/index.ts:308-648](../mcp-task-manager/src/index.ts)
- Agent prompt: [prompts/docker_prompt.md](../prompts/docker_prompt.md)
- Client config: [client.py:59-150](../client.py)

### 4. Container Lifecycle Events

```
Orchestrator.start_session()
  ↓
SandboxManager.create_sandbox(type="docker")
  ↓
DockerSandbox.start()
  ├─ Initializer: Remove old → Create new → Setup → Keep running
  └─ Coding: Reuse/restart → Cleanup processes → Keep running
  ↓
Agent session runs (docker exec commands)
  ↓
DockerSandbox.stop()
  └─ Keep container running (for reuse)
```

---

## Configuration

**File:** `.yokeflow.yaml`

```yaml
sandbox:
  type: docker              # "none", "docker", or "e2b"
  docker_image: node:20-slim
  docker_network: bridge
  docker_memory_limit: 2g
  docker_cpu_limit: "2.0"
  docker_ports:
    - "3001:3001"          # Express backend
    - "5173:5173"          # Vite frontend
```

**Sandbox Types:**

| Type | Description | Use Case |
|------|-------------|----------|
| `none` | No isolation (host) | Debugging, local dev |
| `docker` | Container isolation | Production default |
| `e2b` | E2B cloud sandbox | Not implemented |

---

## Troubleshooting

### "Container already exists"

**Symptom:**
```
Conflict. The container name yokeflow-myproject is already in use
```

**Cause:** Previous session crashed without cleanup

**Solution:**
```bash
docker rm -f yokeflow-myproject
```

The code should handle this automatically, but may fail on crashes.

### "Permission denied" on volume mount

**Solution:**
- Docker Desktop: Add project directory to File Sharing settings
- Linux: Check directory ownership
- SELinux: May need `:z` suffix on volume mount

### Agent uses Bash instead of bash_docker

**Debug:**
1. Check `docker_container` passed to `create_client()`
2. Verify `DOCKER_CONTAINER_NAME` env var in MCP server
3. Check system prompt includes Docker guidance
4. Look for `bash_docker` in agent's tool list

**Solution:** Ensure DockerSandbox created and container name passed through orchestrator → client → MCP.

### "Command not found" in container

**Symptom:**
```
/bin/sh: git: not found
/bin/sh: curl: not found
```

**Cause:** `node:20-slim` is minimal image

**Solution:** Dependencies installed automatically in `_setup_container()`. If command missing, add to setup:

```python
async def _setup_container(self):
    setup_commands = [
        "apt-get update -qq",
        "apt-get install -y -qq git curl build-essential python3 python3-pip",
        "npm install -g pnpm npm",
    ]
```

---

## Future Enhancements

### Container Cleanup Automation

**Goal:** Automatically remove containers when projects complete or are deleted.

**Implementation:**

1. **Project completion hook:**
```python
# orchestrator.py - after project complete
async def start_coding_sessions(...):
    if completed_tasks >= total_tasks:
        await db.mark_project_complete(project_id)

        # NEW: Cleanup container
        container_name = f"yokeflow-{project_name}"
        await cleanup_container(container_name)
```

2. **Project deletion hook:**
```python
# orchestrator.py - delete_project()
async def delete_project(self, project_id: UUID) -> bool:
    # ... existing code ...

    # NEW: Cleanup container
    container_name = f"yokeflow-{project_name}"
    await cleanup_container(container_name)

    return True
```

3. **Cleanup utility:**
```python
async def cleanup_container(container_name: str) -> None:
    """Remove Docker container for project."""
    import docker
    client = docker.from_env()

    try:
        container = client.containers.get(container_name)
        container.remove(force=True)
        logger.info(f"Removed container: {container_name}")
    except docker.errors.NotFound:
        pass  # Already removed
```

**Effort:** 1-2 hours

### Cleanup Commands

**Goal:** Utility to remove orphaned containers.

```bash
# Remove containers for deleted projects
python scripts/cleanup_containers.py

# Remove all stopped containers
python scripts/cleanup_containers.py --stopped

# Force remove all (including running)
python scripts/cleanup_containers.py --all --force
```

**Effort:** 2-3 hours

### Resource Monitoring

**Goal:** Track CPU/memory usage per session.

```python
async def get_stats(self) -> Dict[str, Any]:
    """Get container resource usage."""
    container = self.client.containers.get(self.container_id)
    stats = container.stats(stream=False)

    return {
        "cpu_percent": calculate_cpu_percent(stats),
        "memory_mb": stats['memory_stats']['usage'] / 1024 / 1024,
        "network_rx_bytes": stats['networks']['eth0']['rx_bytes'],
    }
```

**Effort:** 4-6 hours

---

## Related Documentation

- [Configuration](configuration.md) - Sandbox configuration options
- [MCP Usage](mcp-usage.md) - MCP tool development
- [Docker Prompt](../prompts/docker_prompt.md) - Agent guidance for Docker
- [Authentication](authentication.md) - Web UI security

---

**Questions or issues?** Check existing containers with `docker ps -a --filter "name=yokeflow"` and clean up manually if needed.
