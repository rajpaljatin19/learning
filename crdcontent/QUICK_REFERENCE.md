# Quick Reference: Kubernetes CRD Concepts

**One-page summary of everything you need to remember.**

---

## What is a CRD?

A **Custom Resource Definition** teaches Kubernetes about a new resource type. It's an API extension that lets you define and manage custom resources like Pods, Deployments, etc.

---

## Core Concepts (One-Liners)

| Concept | Definition |
|---------|-----------|
| **CRD** | Schema that defines a custom resource type |
| **Custom Resource** | An instance of a CRD (like a Pod is an instance of the Pod schema) |
| **Controller** | A program that watches custom resources and implements business logic |
| **Operator** | CRD + Controller + Operational Knowledge, packaged together |
| **API Group** | Namespace for resources (e.g., `example.com`) |
| **Kind** | Resource type name (e.g., `Book`, `Database`) |
| **Version** | API version (e.g., `v1`, `v1beta1`) |
| **Spec** | Desired state (what user specifies) |
| **Status** | Actual state (what controller reports) |
| **Reconciliation** | Making actual state match desired state |
| **Idempotency** | Same action run multiple times = same result |

---

## CRD Manifest Template

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: <plural>.<group>              # e.g., books.example.com
spec:
  group: <api-group>                  # e.g., example.com
  scope: Namespaced                   # or Cluster
  names:
    kind: <Kind>                      # e.g., Book
    plural: <plural>                  # e.g., books
    shortNames:                       # Optional
    - <shortname>                     # e.g., bk
  versions:
  - name: v1
    served: true
    storage: true
    subresources:
      status: {}                      # Enable status subresource
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              <field>:
                type: <string|integer|boolean|array|object>
                # Add validation: minLength, pattern, enum, required, etc.
          status:
            type: object
            properties:
              <field>: <type>
```

---

## Schema Field Types & Validation

### Basic Types
```yaml
string:                 # Text
integer:               # Whole numbers
number:                # Floating point
boolean:               # true/false
array:                 # Lists of items
object:                # Nested structure
```

### Validation Rules
```yaml
minLength: 1
maxLength: 256
minimum: 0
maximum: 100
pattern: '^[a-z]+$'           # Regex
enum: [value1, value2]        # Allowed values
default: value                # Default value
required:                      # At parent level
- field1
- field2
```

---

## Controller Reconciliation Logic

```python
while True:
    event = watch_for_changes()    # Watch for resource changes
    resource = api.get(event.object)
    
    desired = resource.spec         # What user wants
    actual = resource.status        # What exists
    
    if desired != actual:
        take_action(resource)       # Fix mismatch
    
    update_status(resource)         # Report results
```

---

## Reconciliation Loop Diagram

```
Resource spec changes (desired state)
               ↓
  API server notifies controller
               ↓
  Controller reads resource
               ↓
  Compare: spec vs status
               ↓
     Do they match?
      ↙          ↘
    Yes          No
     |            |
    Done    Take action
             (create, delete, update resources)
             |
             Update status
             |
            Loop continues...
```

---

## RBAC Template (Controller Permissions)

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mycontroller

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: mycontroller
rules:
# Read my CRD
- apiGroups: ["example.com"]
  resources: ["resources"]
  verbs: ["get", "list", "watch"]

# Update my CRD status
- apiGroups: ["example.com"]
  resources: ["resources/status"]
  verbs: ["patch", "update"]

# Create events
- apiGroups: [""]
  resources: ["events"]
  verbs: ["create"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: mycontroller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: mycontroller
subjects:
- kind: ServiceAccount
  name: mycontroller
```

---

## kubectl Commands for CRDs

```bash
# Apply CRD
kubectl apply -f crd.yaml

# Check CRD exists
kubectl get crd books.example.com

# List all custom resources
kubectl get books

# Get one resource
kubectl get book my-book

# Show detailed info
kubectl describe book my-book

# Edit resource
kubectl edit book my-book

# Delete resource
kubectl delete book my-book

# Watch for changes
kubectl get books --watch

# Get in YAML format
kubectl get book my-book -o yaml

# Update status (controller normally does this)
kubectl patch book my-book --type merge \
  -p '{"status":{"ready":true}}'

# List with short name
kubectl get bk
```

---

## Common Gotchas

| Issue | Solution |
|-------|----------|
| "Unknown resource" error | CRD not installed; run `kubectl apply -f crd.yaml` |
| Validation failed | Check schema constraints (required, min/max, enum, pattern) |
| Status not updating | Use `status` subresource; controller needs `patch` permission |
| Controller doesn't see changes | Check: RBAC permissions, not filtering correctly, no crash |
| Idempotency issues | Don't increment counters; set values based on current state |
| Long reconciliation time | Split into multiple resources or async operations |

---

## Design Checklist

- [ ] CRD has a clear, single purpose
- [ ] Spec and status are properly separated
- [ ] Schema validation catches invalid input early
- [ ] Controller reconciliation is idempotent
- [ ] Owner references used for child resources
- [ ] RBAC permissions are minimal but sufficient
- [ ] Error handling covers transient + permanent failures
- [ ] Conditions used for complex state
- [ ] Events created for audit trail
- [ ] Monitoring/logging planned
- [ ] Documentation with examples
- [ ] Testing strategy defined

---

## Real-World Examples to Learn From

- **Prometheus Operator**: Manages Prometheus instances
- **ArgoCD**: GitOps controller
- **Cert-Manager**: Manages TLS certificates
- **Redis Operator**: Manages Redis clusters
- **Stateful cluster operators**: Databases, caches, message queues

Find them on: https://operatorhub.io

---

## Learning Resources

| Resource | Purpose |
|----------|---------|
| https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/ | Official CRD documentation |
| https://kubernetes.io/docs/concepts/extend-kubernetes/operator/ | Operator pattern |
| https://sdk.operatorframework.io/ | Operator SDK (tool for building operators) |
| https://github.com/operator-framework/awesome-operators | Curated list of operators |
| https://operatorhub.io | Marketplace of operators |

---

## This Plan's Phases Recap

| Phase | Concept | Outcome |
|-------|---------|---------|
| 1 | **Fundamentals** | Understand what CRDs are & why they exist |
| 2 | **YAML & Validation** | Learn to write CRD manifests with schemas |
| 3 | **Controllers & Reconciliation** | Understand the watch-reconcile loop |
| 4 | **Operators & Best Practices** | Packaging, RBAC, real-world patterns |
| 5 | **Implementation Planning** | Design YOUR specific CRD before coding |

---

## Next: After Learning Plan

1. **Pick a simple domain** (e.g., config management, data processing)
2. **Build a simple CRD** (just spec, no fancy validation initially)
3. **Build a simple controller** (watches & does one thing)
4. **Test thoroughly** (manual testing on local cluster)
5. **Add features incrementally** (validation, error handling, monitoring)
6. **Deploy to production** (when stable)
7. **Gather feedback** (from users, iterate)

Start small. Most operators began as simple prototypes. 🚀

---

## Get Unstuck

- **Confused about terminology?** → Back to Phase 1
- **YAML syntax help?** → Check Phase 2 examples
- **Controller logic unclear?** → Review Phase 3 reconciliation loop
- **Design questions?** → Use Phase 5 specification template
- **Specific error?** → Check "Common Gotchas" above

Good luck! 💪
