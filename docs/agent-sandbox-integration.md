# Agent Sandbox Integration Design

**Status:** Proposal  
**Author:** Eclipse Che Team  
**Date:** 2026-07-28  
**Target Release:** TBD

## Executive Summary

This document proposes integrating [Kubernetes agent-sandbox](https://github.com/kubernetes-sigs/agent-sandbox) into che-operator to expand Eclipse Che's capabilities beyond Cloud Development Environments (CDEs) to also support AI agent runtimes. This would position Eclipse Che as a unified platform for both human developers and AI-powered automation.

**Key Proposal:** Add agent-sandbox as an optional dependency alongside DevWorkspace Operator, enabling Eclipse Che to orchestrate secure, isolated AI agent execution environments while maintaining its core CDE functionality.

## Goals

- Enable Eclipse Che to manage AI agent runtime sandboxes alongside traditional development workspaces
- Provide strong isolation (kernel-level) for executing potentially untrusted LLM-generated code
- Leverage agent-sandbox's hibernation/resume capabilities for efficient resource usage
- Maintain backward compatibility with existing Che installations
- Follow the same integration patterns established with DevWorkspace Operator

## Non-Goals

- Replace or modify DevWorkspace Operator functionality
- Build LLM/AI capabilities into che-operator itself (only provide runtime infrastructure)
- Support agent-sandbox as the exclusive workspace runtime (DWO remains primary for CDEs)
- Implement agent orchestration logic (agent coordination, planning, etc.)

## Background

### Current Architecture

Eclipse Che currently supports CDEs through integration with DevWorkspace Operator (DWO):

```
┌─────────────────────────────────────────────────────────┐
│                     Eclipse Che                          │
│  ┌──────────────┐  ┌─────────┐  ┌──────────────┐       │
│  │  Dashboard   │  │ Gateway │  │ DevFile Reg │  ...   │
│  └──────────────┘  └─────────┘  └──────────────┘       │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│            DevWorkspace Operator (DWO)                   │
│  - Manages workspace pod lifecycle                       │
│  - Devfile-based configuration                          │
│  - Routing via CheRoutingSolver                         │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
                   User Workspaces
                   (CDEs for humans)
```

### DevWorkspace vs Agent Sandbox

| Aspect | DevWorkspace Operator | Agent Sandbox |
|--------|----------------------|---------------|
| **Primary Use Case** | Human development environments | AI agent runtimes, untrusted code execution |
| **Configuration** | Devfile (YAML) | PodTemplate |
| **Isolation** | Standard container isolation | Strong isolation (gVisor, Kata) via RuntimeClass |
| **State Management** | Persistent volumes | Persistent volumes + deep hibernation |
| **Multi-container** | Yes (sidecar pattern) | No (single container focus) |
| **Lifecycle** | Start/stop | Start/stop/hibernate/resume |
| **Warm Pools** | No | Yes (SandboxWarmPool for fast allocation) |
| **Target Tenant** | Authenticated users | Programmatic agents/services |

### Why Agent Sandbox?

**Stronger Isolation for Untrusted Code:**  
AI agents may execute LLM-generated code that requires kernel-level isolation beyond standard containers.

**Resource Efficiency:**  
Hibernation/resume and warm pools optimize costs for intermittent agent workloads.

**Stable Identity:**  
Long-running agents benefit from persistent network identity across restarts.

**Programmatic API:**  
Designed for applications to consume programmatically, not just human users.

## Proposed Architecture

### High-Level Design

```
┌─────────────────────────────────────────────────────────────────┐
│                        Eclipse Che                               │
│  ┌──────────────┐  ┌─────────┐  ┌──────────────┐               │
│  │  Dashboard   │  │ Gateway │  │ DevFile Reg │  ...            │
│  └──────────────┘  └─────────┘  └──────────────┘               │
│                                                                  │
│  New: Agent Dashboard / API for managing agent sandboxes        │
└─────────────────────────────────────────────────────────────────┘
            │                                    │
            ▼                                    ▼
┌──────────────────────────┐      ┌──────────────────────────────┐
│ DevWorkspace Operator    │      │   Agent Sandbox              │
│ - CDEs (devfile-based)   │      │   - AI agent runtimes        │
│ - Human developers       │      │   - Strong isolation         │
│ - Multi-container        │      │   - Hibernation/resume       │
└──────────────────────────┘      └──────────────────────────────┘
            │                                    │
            ▼                                    ▼
    User Workspaces                      Agent Sandboxes
    (CDEs for humans)              (Isolated agent execution)
```

### Integration Levels (Phased Approach)

#### Phase 1: Coexistence (Minimal Integration)

**Goal:** Verify agent-sandbox can run alongside che-operator without conflicts.

**Scope:**
- Deploy agent-sandbox operator in cluster
- No changes to che-operator code
- Manual `Sandbox` CR creation by advanced users
- Validate networking, RBAC, resource quotas don't conflict

**Deliverables:**
- Installation documentation
- Compatibility matrix

**Risk:** Low  
**Effort:** 1-2 weeks

#### Phase 2: API Integration (Recommended Initial Target)

**Goal:** Add first-class support for agent runtimes in CheCluster API.

**Scope:**
- Extend `CheCluster` CR with `spec.Components.AgentRuntimes` section
- Add controller to watch agent-sandbox namespace
- Integrate agent-sandbox routing with Che gateway (if needed)
- Add RBAC management for agent sandboxes
- Dashboard UI to view (read-only) agent sandboxes

**API Changes:**
```go
type CheClusterSpec struct {
    // ... existing fields
    
    // AgentRuntimes configures AI agent execution environments
    // +optional
    AgentRuntimes *AgentRuntimesConfiguration `json:"agentRuntimes,omitempty"`
}

type AgentRuntimesConfiguration struct {
    // Enable agent runtime support
    // +optional
    // +kubebuilder:default:=false
    Enable bool `json:"enable"`
    
    // Namespace where agent-sandbox operator is deployed
    // +optional
    // +kubebuilder:default:="agent-sandbox-system"
    Namespace string `json:"namespace,omitempty"`
    
    // Default RuntimeClass for agent sandboxes (e.g., "gvisor", "kata")
    // +optional
    RuntimeClassName string `json:"runtimeClassName,omitempty"`
    
    // WarmPool configuration for pre-warming agent sandboxes
    // +optional
    WarmPool *WarmPoolConfig `json:"warmPool,omitempty"`
    
    // DefaultResources for agent sandboxes
    // +optional
    Resources *ResourceRequirements `json:"resources,omitempty"`
}

type WarmPoolConfig struct {
    // Enable warm pool for faster agent startup
    // +optional
    Enable bool `json:"enable"`
    
    // Number of pre-warmed sandboxes to maintain
    // +optional
    // +kubebuilder:default:=5
    Size int `json:"size,omitempty"`
    
    // Template reference for warm pool sandboxes
    // +optional
    TemplateRef string `json:"templateRef,omitempty"`
}
```

**Deliverables:**
- CRD changes + regenerated manifests
- `AgentRuntimeReconciler` controller
- RBAC rules for managing `Sandbox` CRs
- Documentation for enabling agent runtimes

**Risk:** Medium  
**Effort:** 4-6 weeks

#### Phase 3: Full Integration (Future)

**Goal:** Unified user experience for CDEs and agent runtimes.

**Scope:**
- Dashboard UI for creating/managing agent sandboxes
- Unified routing through Che gateway
- Integration with Che authentication/authorization
- Templates for common agent types (code execution, testing, build)
- Metrics and monitoring for agent sandboxes
- Agent-to-workspace communication patterns

**Deliverables:**
- Full dashboard support
- Che CLI commands for agent management
- Production-ready warm pool management
- SLA monitoring

**Risk:** High  
**Effort:** 12-16 weeks

## Component Design (Phase 2)

### New Controller: AgentRuntimeReconciler

Similar to how che-operator integrates with DWO via `DevWorkspaceRouting` solver, add a new controller.

**Location:** `controllers/agentruntime/agentruntime_controller.go`

**Responsibilities:**
1. Watch `CheCluster.spec.agentRuntimes` configuration
2. Ensure agent-sandbox operator is present (health check)
3. Create default `SandboxTemplate` CRs for common agent types
4. Manage `SandboxWarmPool` if enabled
5. Set up RBAC for Che users to create `Sandbox` CRs
6. Configure routing integration (if required)

**Reconciliation Logic:**
```go
type AgentRuntimeReconciler struct {
    // Implements Reconcilable interface
}

func (r *AgentRuntimeReconciler) Reconcile(ctx *chetypes.DeployContext) (reconcile.Result, bool, error) {
    if ctx.CheCluster.Spec.AgentRuntimes == nil || !ctx.CheCluster.Spec.AgentRuntimes.Enable {
        // Agent runtimes disabled, skip
        return reconcile.Result{}, true, nil
    }
    
    // 1. Verify agent-sandbox operator is running
    if err := r.ensureAgentSandboxOperator(ctx); err != nil {
        return reconcile.Result{}, false, fmt.Errorf("agent-sandbox operator not ready: %w", err)
    }
    
    // 2. Create default SandboxTemplates
    if err := r.syncSandboxTemplates(ctx); err != nil {
        return reconcile.Result{}, false, fmt.Errorf("failed to sync sandbox templates: %w", err)
    }
    
    // 3. Configure WarmPool if enabled
    if ctx.CheCluster.Spec.AgentRuntimes.WarmPool != nil && ctx.CheCluster.Spec.AgentRuntimes.WarmPool.Enable {
        if err := r.syncWarmPool(ctx); err != nil {
            return reconcile.Result{}, false, fmt.Errorf("failed to sync warm pool: %w", err)
        }
    }
    
    // 4. Set up RBAC for agent sandboxes
    if err := r.syncRBAC(ctx); err != nil {
        return reconcile.Result{}, false, fmt.Errorf("failed to sync RBAC: %w", err)
    }
    
    return reconcile.Result{}, true, nil
}
```

### DeployContext Extensions

Add to `pkg/common/chetypes/types.go`:

```go
type DeployContext struct {
    // ... existing fields
    
    // AgentSandboxNamespace is the namespace where agent-sandbox operator runs
    AgentSandboxNamespace string
    
    // AgentRuntimesEnabled indicates if agent runtime support is active
    AgentRuntimesEnabled bool
}
```

### Default SandboxTemplates

Create default templates in `pkg/deploy/agentruntime/templates/`:

**1. Code Execution Agent**
```yaml
apiVersion: agents.x-k8s.io/v1beta1
kind: SandboxTemplate
metadata:
  name: che-code-execution-agent
  namespace: {{ .Namespace }}
spec:
  sandboxTemplate:
    spec:
      podTemplate:
        spec:
          runtimeClassName: {{ .RuntimeClassName }}
          containers:
          - name: agent
            image: {{ .AgentImage }}
            resources:
              requests:
                cpu: "100m"
                memory: "256Mi"
              limits:
                cpu: "2000m"
                memory: "2Gi"
```

**2. Testing Agent**
**3. Build Agent**

### RBAC Configuration

Create role/rolebinding to allow Che users to manage sandboxes:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: che-agent-sandbox-user
rules:
- apiGroups: ["agents.x-k8s.io"]
  resources: ["sandboxes", "sandboxclaims"]
  verbs: ["create", "get", "list", "watch", "delete"]
- apiGroups: ["agents.x-k8s.io"]
  resources: ["sandboxtemplates"]
  verbs: ["get", "list"]
```

### Routing Integration

**Option A: No Integration (Phase 2)**  
Agent sandboxes expose services independently, accessed via cluster DNS.

**Option B: Gateway Integration (Phase 3)**  
Extend Che gateway to route requests to agent sandboxes similar to workspace routing:
- `https://che-host/agent/<sandbox-name>/` → agent sandbox service
- Reuse authentication from Che gateway
- May require implementing an `AgentSandboxRoutingSolver`

**Recommendation:** Start with Option A for Phase 2.

## Security Considerations

### Isolation Requirements

**Threat Model:**  
AI agents execute potentially untrusted LLM-generated code that could:
- Attempt privilege escalation
- Exfiltrate data from cluster
- Consume excessive resources (DoS)
- Escape container boundaries

**Mitigation:**
1. **Mandatory RuntimeClass:** Require gVisor or Kata for all agent sandboxes
2. **NetworkPolicy:** Default deny-all, explicit allow only required connections
3. **ResourceQuota:** Hard limits on CPU/memory per sandbox
4. **PodSecurityPolicy/Standards:** Enforce restrictive profiles
5. **RBAC:** Limit agent sandbox permissions strictly

### Configuration

```go
type AgentRuntimesConfiguration struct {
    // ... existing fields
    
    // Security configuration for agent sandboxes
    // +optional
    Security *AgentSecurityConfig `json:"security,omitempty"`
}

type AgentSecurityConfig struct {
    // Require RuntimeClass (cannot be empty)
    // +kubebuilder:validation:Required
    EnforceRuntimeClass bool `json:"enforceRuntimeClass"`
    
    // NetworkPolicy to apply to agent sandboxes
    // +optional
    NetworkPolicy string `json:"networkPolicy,omitempty"`
    
    // PodSecurityStandard level (restricted, baseline, privileged)
    // +optional
    // +kubebuilder:default:="restricted"
    PodSecurityStandard string `json:"podSecurityStandard,omitempty"`
}
```

### Audit Logging

Log all agent sandbox lifecycle events:
- Creation, deletion, hibernation, resume
- Code execution requests
- Network policy violations
- Resource limit breaches

## Installation & Dependencies

### Dependency Management

**Option 1: Optional Dependency (Recommended)**  
Agent-sandbox is optional, installed separately by cluster admin:

```yaml
# CheCluster with agent runtimes
apiVersion: org.eclipse.che/v2
kind: CheCluster
metadata:
  name: eclipse-che
spec:
  agentRuntimes:
    enable: true
    namespace: agent-sandbox-system
    runtimeClassName: gvisor
```

If `agentRuntimes.enable=true` but agent-sandbox operator is not present, reconciliation logs a warning and skips agent runtime setup.

**Option 2: Bundled Dependency**  
Include agent-sandbox in Che's OLM bundle as a dependency.

**Recommendation:** Option 1 for Phase 2 to avoid coupling release cycles.

### Installation Documentation

Provide clear docs for:
1. Installing agent-sandbox operator
2. Configuring RuntimeClass (gVisor/Kata)
3. Enabling agent runtimes in CheCluster
4. Creating first agent sandbox

## Observability

### Metrics

Expose Prometheus metrics:
- `che_agent_sandboxes_total` — total sandboxes managed
- `che_agent_sandboxes_active` — currently running
- `che_agent_sandboxes_hibernated` — in hibernation
- `che_agent_sandbox_creation_duration_seconds` — time to create sandbox
- `che_agent_warmpool_size` — warm pool current size

### Dashboard Integration

**Phase 2 (Read-Only):**
- List agent sandboxes
- Show status (running, hibernated, failed)
- View logs
- Display resource usage

**Phase 3 (Full Management):**
- Create agent sandboxes from templates
- Start/stop/hibernate manually
- Configure warm pools
- View metrics/analytics

## Migration & Rollout

### Rollout Plan

1. **Upstream First:** Merge into che-operator `main` behind feature flag
2. **Preview Release:** Announce in community, gather feedback
3. **Stable Release:** Promote to stable channel after 2+ minor versions
4. **Documentation:** Full guides, tutorials, best practices

### Feature Flag

Control via environment variable initially:

```yaml
env:
- name: ENABLE_AGENT_RUNTIMES
  value: "true"
```

Later promote to CheCluster API.

### Backward Compatibility

- Agent runtime support is **opt-in** via `spec.agentRuntimes.enable`
- Existing CheCluster CRs without this field continue working unchanged
- No impact on DevWorkspace Operator integration

## Testing Strategy

### Unit Tests
- `AgentRuntimeReconciler` logic
- API validation (CRD webhooks)
- RBAC generation

### Integration Tests
- Deploy agent-sandbox + che-operator on Minikube
- Create `Sandbox` CR via CheCluster configuration
- Verify routing (if integrated)
- Test warm pool behavior

### E2E Tests
- Full user journey: enable agent runtimes → create sandbox → execute code → verify isolation
- Security: attempt container escape, verify blocked
- Performance: measure warm pool speedup

## Open Questions

1. **Do we need unified routing?**  
   Should agent sandboxes be accessible via `che-host/agent/...` or is direct service access sufficient?

2. **Authentication model?**  
   Should agents use Che user authentication, service accounts, or API keys?

3. **Agent orchestration?**  
   Does che-operator need to coordinate multiple agents, or is that external (e.g., LangChain, AutoGPT)?

4. **Devfile integration?**  
   Could we extend devfile spec to declare agent requirements alongside CDEs?

5. **Multi-agent communication?**  
   Do we need service mesh integration for agent-to-agent communication?

6. **Billing/quotas?**  
   How do we attribute agent sandbox resource usage to users/teams?

## Alternatives Considered

### Alternative 1: Extend DevWorkspace Operator

**Approach:** Add agent runtime capabilities directly to DWO instead of using agent-sandbox.

**Pros:**
- Single dependency
- Reuse devfile format

**Cons:**
- DWO is focused on CDEs, agent runtimes are architecturally different
- Would require upstream DWO changes (slower adoption)
- Mixing concerns in one operator

**Decision:** Rejected. Keep concerns separated.

### Alternative 2: Build Custom Agent Runtime

**Approach:** Implement agent sandbox functionality directly in che-operator.

**Pros:**
- Full control over implementation
- Tight integration with Che

**Cons:**
- Duplicate work that agent-sandbox already provides
- Higher maintenance burden
- Misses community benefits of Kubernetes SIG project

**Decision:** Rejected. Leverage existing community project.

### Alternative 3: No Agent Support

**Approach:** Keep che-operator focused solely on CDEs.

**Pros:**
- Simpler architecture
- Clear scope

**Cons:**
- Misses opportunity to support AI-augmented development
- Users will build workarounds anyway

**Decision:** Rejected. Market demand for AI agent infrastructure is growing.

## Success Metrics

- **Adoption:** % of Che installations with agent runtimes enabled
- **Performance:** Agent sandbox creation time < 5s with warm pools
- **Security:** Zero container escape incidents
- **Reliability:** Agent sandbox uptime > 99.9%
- **Community:** Contributions/integrations from ecosystem partners

## References

- [agent-sandbox GitHub](https://github.com/kubernetes-sigs/agent-sandbox)
- [DevWorkspace Operator](https://github.com/devfile/devworkspace-operator)
- [Eclipse Che Architecture](https://eclipse.dev/che/docs/)
- [gVisor](https://gvisor.dev/)
- [Kata Containers](https://katacontainers.io/)

## Appendix: Example Usage

### Creating an Agent Sandbox via Che API

```bash
# Enable agent runtimes in CheCluster
kubectl patch checluster/eclipse-che \
  --type merge \
  -p '{"spec":{"agentRuntimes":{"enable":true}}}'

# Create a sandbox from template
kubectl apply -f - <<EOF
apiVersion: agents.x-k8s.io/v1beta1
kind: SandboxClaim
metadata:
  name: my-code-agent
  namespace: my-namespace
spec:
  templateRef: che-code-execution-agent
EOF

# Check status
kubectl get sandbox my-code-agent -o yaml
```

### Accessing Agent Sandbox

```bash
# Port-forward to agent sandbox
kubectl port-forward sandbox/my-code-agent 8080:8080

# Execute code (example agent API)
curl -X POST http://localhost:8080/execute \
  -d '{"code": "print(\"Hello from agent!\")"}'
```

---

**Next Steps:**
1. Review this proposal with Eclipse Che community
2. Prototype Phase 1 (coexistence testing)
3. Gather feedback from users interested in AI agent use cases
4. Refine API design based on feedback
5. Implement Phase 2 if approved
