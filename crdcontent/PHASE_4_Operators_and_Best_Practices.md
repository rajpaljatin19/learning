# Phase 4: Operators & Best Practices

You now understand the full picture: CRDs define resources, controllers implement behavior. 

This phase covers how to **package and deploy controllers** in production, and shows you **proven patterns** from the Kubernetes ecosystem.

---

## What is an Operator?

### Definition

An **Operator** is a package that includes:
1. **CRD(s)** - Resource definitions
2. **Controller(s)** - Business logic
3. **RBAC** - Permissions the controller needs
4. **Operational knowledge** - Deployment guides, examples, docs

**Equation**: `Operator = CRD + Controller + Operational Knowledge`

### Analogy: A Database Administrator's Knowledge, Automated

Before Kubernetes, you'd hire a DBA to manage databases:
- Create new databases
- Backup and restore
- Perform upgrades
- Monitor health
- Troubleshoot issues

An **Operator** codifies that knowledge into automation:
- `DatabaseCRD` = The resource
- `DatabaseController` = The automation (what the DBA knows)
- Together = An operator that manages databases automatically

---

## CRD vs. Operator: The Difference

| Aspect | CRD | Operator |
|--------|-----|----------|
| **What is it?** | Resource definition (schema) | CRD + Controller + Package |
| **Does it do anything alone?** | No, just defines structure | Yes, actively manages resources |
| **Example** | `Book` CRD definition | Book Operator (CRD + controller that enforces rules) |
| **Installed by** | Cluster admin | Cluster admin |
| **Used by** | Developers (create instances) | System (controller runs autonomously) |

---

## Operator Architecture

### Typical Operator Components

```
Book Operator Package (Helm Chart / Operator Bundle)
│
├─ CRD Manifest
│  └─ books.example.com CustomResourceDefinition
│
├─ Controller Deployment
│  └─ Runs as a Pod in kube-system or operator namespace
│
├─ RBAC (Service Account, Role, RoleBinding)
│  └─ Permissions: read pooks, write status, create events
│
├─ Values/Config
│  └─ Controller configuration (replicas, resources, logging)
│
└─ Documentation & Examples
   └─ How to use the operator
```

### Deployment Flow

```
1. Operator Developer:
   - Designs CRD
   - Builds controller code
   - Packages as Helm chart / Operator bundle

2. Cluster Admin:
   - Installs operator: helm install book-operator ./chart
   
3. Controller Pod Starts:
   - Reads CRD from API server
   - Starts watching for Book changes
   
4. Developer:
   - kubectl apply -f book.yaml
   - Creates a Book instance
   
5. Controller Reconciles:
   - Sees the new Book
   - Takes action based on spec
   - Updates status
   
6. Developer:
   - Uses the managed resource
   - Kubernetes automation handles the rest
```

---

## RBAC: What Permissions Do Controllers Need?

Controllers can't just do anything. They need explicit permissions.

### Example: Book Controller RBAC

```yaml
---
# Service Account for the controller
apiVersion: v1
kind: ServiceAccount
metadata:
  name: book-controller
  namespace: book-system

---
# Role: What can the controller do?
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: book-controller
  namespace: book-system
rules:
# Read Book CRD instances
- apiGroups: ["example.com"]
  resources: ["books"]
  verbs: ["get", "list", "watch"]

# Update Book status
- apiGroups: ["example.com"]
  resources: ["books/status"]
  verbs: ["get", "patch", "update"]

# Create events (for logging)
- apiGroups: [""]
  resources: ["events"]
  verbs: ["create", "patch"]

# If the controller needs to create ConfigMaps or Secrets
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update"]

---
# RoleBinding: Attach the role to the service account
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: book-controller
  namespace: book-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: book-controller
subjects:
- kind: ServiceAccount
  name: book-controller
  namespace: book-system

---
# Controller Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: book-controller
  namespace: book-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: book-controller
  template:
    metadata:
      labels:
        app: book-controller
    spec:
      serviceAccountName: book-controller  # ← Use the service account
      containers:
      - name: controller
        image: example.com/book-controller:v1.0.0
        imagePullPolicy: IfNotPresent
```

**Key points**:
- Service Account: Identity for the controller Pod
- Role: Permissions (what resources, what verbs)
- RoleBinding: Connects service account to role
- Deployment: The controller Pod itself

---

## Best Practices for CRD Design

### 1. Single Responsibility

Each CRD should have **one clear purpose**.

**Good**:
```yaml
# Database CRD manages databases
kind: Database
spec:
  name: production-db
  size: 10Gi
  version: 13.0
```

**Bad**:
```yaml
# CRD that's a kitchen sink
kind: MegaResource
spec:
  databaseName: db
  webServerConfig: ...
  loadBalancerSettings: ...
  monitoringRules: ...
```

**Why?** Single responsibility makes CRDs:
- Easier to reason about
- Easier to reuse
- Easier to test
- Easier to extend

### 2. Separate Concerns: Spec vs. Status

**Spec** = What the user wants (desired state)  
**Status** = What actually exists (actual state)

**Good**:
```yaml
spec:
  replicas: 3           # User wants 3 replicas
status:
  ready: 2              # But only 2 are ready
  conditions:
  - type: Ready
    status: "False"
    reason: "OnePodCrashing"
```

The controller reconciles the difference.

**Bad** (mixing them):
```yaml
spec:
  name: db
  actualDiskUsage: 5Gi  # ❌ Actual state in spec
  activeConnections: 12 # ❌ Actual state in spec
```

Controllers should update `status`, never the actual `spec`.

### 3. Idempotent Reconciliation

Your controller logic should be idempotent:

```python
# Good: idempotent
def reconcile(book):
    if book.spec.available and book.status.borrowed:
        book.status.issue = "Conflict"  # Same every time
    api.update_status(book)

# Bad: not idempotent
def reconcile(book):
    book.status.checkouts += 1  # Increments each time!
    api.update_status(book)
```

### 4. Use Conditions for Complex State

For multi-faceted status:

```yaml
status:
  conditions:
  - type: Ready
    status: "True"
    reason: "AllPodsReady"
    message: "All replicas are running"
  - type: HighMemory
    status: "False"
    reason: "MemoryNormal"
  - type: FailedDependency
    status: "True"
    reason: "MissingSecret"
    message: "Secret 'db-password' not found"
```

Tools can query these conditions instead of string-matching status fields.

### 5. Validation at the CRD Level

Validate as much as possible in the CRD schema. Don't let invalid data get stored.

```yaml
properties:
  replicas:
    type: integer
    minimum: 1            # ✓ Validated here
    maximum: 100
  image:
    type: string
    pattern: '.*:.*'      # ✓ Simple regex validation
```

This prevents:
- Bad data in etcd
- Controller crashes from unexpected input
- Debugging issues in the controller instead of the CRD

### 6. Proper Error Handling in Controllers

```python
def reconcile(book):
    try:
        # Optimistic: try the action
        take_action(book)
        book.status.error = ""
        api.update_status(book)
    
    except TransientError as e:
        # Network blip, etc. Kubernetes will retry.
        logger.warning(f"Transient error: {e}")
        # Don't update status, just return
        raise  # Kubernetes requeues automatically
    
    except PermanentError as e:
        # Won't succeed no matter how many retries
        book.status.error = str(e)
        book.status.errorCount += 1
        api.update_status(book)
        # Don't raise; mark as processed but failed
```

### 7. Use Events for Audit Trail

Record important actions:

```python
def reconcile(book):
    # ... reconciliation logic ...
    
    # Record an event for auditability
    event = Event(
        type="Normal",  # or "Warning"
        reason="BookBorrowed",
        message=f"Book borrowed by {borrower}",
        object=book
    )
    api.create_event(event)
```

Users can see: `kubectl describe book my-book`

### 8. Version Your CRDs

Plan for evolution:

```yaml
versions:
- name: v1beta1
  served: true
  storage: false        # Old version, no longer stores data
- name: v1
  served: true
  storage: true         # New version, stores data
  conversion:
    strategy: None      # Or implement conversion webhook
```

This allows graceful upgrades without losing data.

---

## Common Operator Patterns in the Wild

### Pattern 1: The Prometheus Operator

**What it does**: Manages Prometheus monitoring instances

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: my-prometheus
spec:
  replicas: 3
  retention: 24h
  serviceMonitorSelector:
    matchLabels:
      prometheus: enabled
```

**Controller logic**:
- Watches Prometheus CRD instances
- Creates StatefulSets (actually runs Prometheus)
- Manages configuration
- Handles upgrades

**Lesson**: CRDs can manage complex applications by creating lower-level Kubernetes resources.

### Pattern 2: The Operator Framework

Tools like **Operator SDK** help you build operators:

```bash
# Generate a new operator
operator-sdk new database-operator --repo github.com/example/database-operator
operator-sdk add api --api-version example.com/v1alpha1 --kind Database
operator-sdk add controller --api-version example.com/v1alpha1 --kind Database
```

**Lesson**: These tools abstract away boilerplate so you focus on business logic.

### Pattern 3: Operator Hub

A marketplace for discovering operators:
- Prometheus Operator
- Kasten K10 (backup/restore)
- Elasticsearch Operator
- Redis Operator
- Many more!

**Lesson**: Operators are the standard way to extend Kubernetes.

---

## When NOT to Use CRDs

CRDs are powerful, but not always the right tool:

| Use CRD | Don't Use CRD |
|---------|--------------|
| Managing application instances (Databases, Caches) | One-off scripts or one-time setup |
| Complex multi-step operations | Simple data that doesn't need reconciliation |
| Multi-tenant systems | Single-tenant internal tools |
| Resources that need RBAC control | No need for permission model |
| Systems that benefit from audit trail | Throwaway tooling |

---

## Hands-On: Design Your Operator

Before implementing, plan it out:

### Step 1: Define Your Resource

**My CRD will manage**: `_____________________`

**Example**: A `Blog` CRD that manages blog websites.

### Step 2: Sketch the Manifest

**What fields does your resource need?**

```yaml
apiVersion: example.com/v1
kind: Blog
metadata:
  name: my-blog
spec:
  title: "My Tech Blog"
  theme: "dark"
  domain: "blog.example.com"
  # ... other fields?
```

### Step 3: Outline Reconciliation Logic

**What should the controller do when this resource changes?**

- [ ] Create a web server deployment?
- [ ] Register DNS?
- [ ] Set up SSL certificates?
- [ ] Install themes or plugins?

### Step 4: Plan Status

**What should the status show?**

```yaml
status:
  conditions:
  - type: DeploymentReady
    status: "True"
  - type: DNSConfigured
    status: "True"
  - type: SSLCertificateReady
    status: "False"
    reason: "CertPending"
  deploymentName: "blog-my-blog"
  publicURL: "http://blog.example.com"
```

### Step 5: Identify Dependencies

**What external resources or APIs does your controller need?**

- [ ] External database?
- [ ] DNS provider API?
- [ ] Certificate issuer (Let's Encrypt)?
- [ ] Container registry credentials?

---

## Key Takeaways

✅ Operators package CRDs + Controllers + Operational Knowledge  
✅ Controllers need RBAC permissions to manage resources  
✅ Single responsibility per CRD makes them easier to reason about  
✅ Separate spec (desired) from status (actual)  
✅ Reconciliation must be idempotent  
✅ Use conditions for complex multi-faceted state  
✅ Validate early in the CRD, not the controller  
✅ Handle transient and permanent errors differently  
✅ Use events for audit trails  

---

## Next: Phase 5

In Phase 5, you'll consolidate everything and create a **specification document for YOUR custom CRD**—ready to implement.

**Ready?** Proceed to `PHASE_5_Implementation_Planning.md`
