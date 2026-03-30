# Phase 5: Implementation Planning

You've learned the concepts. Now let's design **your specific CRD** before writing any code.

This phase is a **specification document** for your custom resource. Having a clear plan prevents mistakes, wasted effort, and architectural redesigns later.

---

## The Planning Process

### Step 1: Define the Problem & Use Case

### Step 2: Design the Resource Schema

### Step 3: Plan Reconciliation Logic

### Step 4: Identify Implementation Dependencies

### Step 5: Review & Validate

---

## TEMPLATE: Your CRD Specification

Use this template to flesh out your CRD. Answer each question thoroughly.

---

# `<RESOURCE_NAME>` CRD Specification

## 1. Problem Statement

### What problem does this CRD solve?

Write 2-3 sentences explaining the business problem.

**Example**:
> We run multiple microservices that each need their own PostgreSQL database. Currently, database provisioning is manual and error-prone. This CRD automates database provisioning, backup scheduling, and lifecycle management.

**Your answer**:
```
[Answer here]
```

### Who will use this CRD?

- [ ] Application developers (create resources)
- [ ] Platform engineers (manage operators)
- [ ] Cluster administrators (manage permissions)
- [ ] End users (indirectly, through the app)

---

## 2. Resource Design

### API Group & Kind

**API Group** (reverse-domain format):
```
example.com
```

**Kind** (PascalCase):
```
Database
```

**Full Resource Name**:
```
databases.example.com/v1
```

### Scope

- [ ] **Namespaced** (lives in a namespace, most resources)
- [ ] **Cluster** (cluster-wide, like Nodes)

**Rationale**: 
```
[Why did you choose this?]
```

### Schema: Spec Fields

Define what the user specifies (desired state).

**Format**: Complete the table below for each field.

| Field Name | Type | Required? | Validation | Default | Description |
|-----------|------|-----------|-----------|---------|-------------|
| `name` | string | Yes | minLength: 1, maxLength: 63 | - | Name of the database |
| `size` | string | Yes | enum: ["small", "medium", "large"] | - | Database size tier |
| `backupSchedule` | string | No | pattern: Cron expression | `"0 2 * * *"` | When to backup (cron) |
| | | | | | |

### Schema: Status Fields

Define what the controller reports (actual state).

| Field Name | Type | Description | Who sets it? |
|-----------|------|-------------|-------------|
| `ready` | boolean | Is the database ready? | Controller |
| `endpointURL` | string | Connection string | Controller |
| `backupCount` | integer | Total backups taken | Controller |
| `conditions` | array | Complex status | Controller |

### Example Manifest (Populated)

How will a developer use this CRD?

```yaml
apiVersion: example.com/v1
kind: Database
metadata:
  name: production-db
  namespace: default
spec:
  # Desired state
  version: "13.0"
  size: "large"
  backupSchedule: "0 2 * * *"
  storageSize: "100Gi"

status:
  # Actual state (set by controller)
  ready: true
  endpointURL: "postgres://production-db:5432/myapp"
  conditions:
  - type: Ready
    status: "True"
    reason: "AllComponentsReady"
  - type: Backup
    status: "True"
    reason: "LastBackupSuccessful"
```

---

## 3. Reconciliation Logic

### What does the controller do?

Describe the reconciliation loop. What happens when a resource changes?

**Format**: Event → Action

#### Event 1: Resource Created
```
When: Database is created with size="large"
Controller:
  1. Create a StatefulSet with large resource requests
  2. Create a PersistentVolumeClaim for storage
  3. Create a Secret for database password
  4. Wait for StatefulSet to be ready
  5. Update status.endpointURL with the database URL
  6. Set status.ready = true
```

#### Event 2: Resource Updated (size changed)
```
When: spec.size changes from "small" to "large"
Controller:
  1. Scale up the StatefulSet replicas if needed
  2. Update volume size
  3. Coordinate downtime-free upgrade
  4. Set status.ready = true when complete
```

#### Event 3: Resource Deleted
```
When: Database is deleted
Controller:
  1. Take a final backup
  2. Delete the StatefulSet
  3. Delete PersistentVolumeClaim
  4. Clean up secrets
  5. Log that database was deleted (audit trail)
```

### Error Handling

What can go wrong? How does the controller handle it?

| Scenario | Error Type | Recovery |
|----------|-----------|----------|
| PVC can't be created (no space) | Permanent | Set condition, alert user, ask for cleanup |
| Network timeout during creation | Transient | Retry automatically (Kubernetes does this) |
| Invalid backup schedule format | Permanent | Set `.status.error`, never attempt backup |
| StatefulSet Pod crashes | Transient | Retry, log in events, user should investigate |

---

## 4. Resource Management

### Child Resources

What resources does your controller create/manage?

```
Database CRD
  ├─ StatefulSet (Runs PostgreSQL)
  ├─ PersistentVolumeClaim (Storage)
  ├─ Service (DNS name for database)
  └─ ConfigMap (Database config)
```

**Ownership**: All child resources should have owner references pointing back to the Database CRD. When the Database is deleted, all children are cascade-deleted.

### Configuration Storage

Where does your controller read configuration?

- [ ] ConfigMaps
- [ ] Secrets
- [ ] Custom annotations on the CRD
- [ ] Environment variables

### External Dependencies

Does your controller need to call external APIs?

- [ ] DNS provider (e.g., Route53)
- [ ] Certificate issuer (e.g., Let's Encrypt)
- [ ] Cloud provider API (e.g., AWS, GCP)
- [ ] Message queue
- [ ] Other

**For each dependency, answer**:
- How is it authenticated?
- What happens if it's unavailable?
- Can reconciliation proceed without it?

---

## 5. RBAC & Permissions

### Service Account

The controller runs as a Service Account. What permissions does it need?

```yaml
rules:
# [List every resource and verb the controller needs]
- apiGroups: ["example.com"]
  resources: ["databases"]
  verbs: ["get", "list", "watch"]

- apiGroups: ["example.com"]
  resources: ["databases/status"]
  verbs: ["get", "patch", "update"]

- apiGroups: ["apps"]
  resources: ["statefulsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]

- apiGroups: [""]
  resources: ["persistentvolumeclaims"]
  verbs: ["get", "list", "watch", "create", "delete"]

- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list", "watch", "create"]

- apiGroups: [""]
  resources: ["events"]
  verbs: ["create", "patch"]
```

### User Permissions

What should *developers* be allowed to do?

- [ ] Create databases (write `databases`)
- [ ] List databases (list `databases`)
- [ ] Update databases (patch `databases`)
- [ ] Modify status? (No, controller does this)
- [ ] Delete databases (delete `databases`)

---

## 6. Versioning & Evolution

### Initial Version

```
API Version: example.com/v1 (or v1alpha1 if experimental)
Status: Stable (or Alpha/Beta)
```

### Future Versions

Plan for evolution. What might change?

```
v1beta1 → v1: Add field X, remove deprecated field Y, change field Z type
v1 → v2: Major redesign of spec structure (requires migration webhook)
```

---

## 7. Validation Rules

### CRD Validation

What validations should happen at the API level (CRD schema)?

```yaml
properties:
  spec:
    required:
    - name
    - size
    properties:
      name:
        type: string
        minLength: 1
        maxLength: 63
        pattern: '^[a-z0-9\-]+$'  # Only lowercase, digits, hyphens
      
      size:
        type: string
        enum: ["small", "medium", "large", "xlarge"]
      
      backupSchedule:
        type: string
        pattern: '^((((\*|(([0-9]|1[0-9]|2[0-9]|3[0-9]|4[0-9]|5[0-9])\/([0-9]|1[0-9]|2[0-9]|3[0-9]|4[0-9]|5[0-9]))|(([0-9]|1[0-9]|2[0-9]|3[0-9]|4[0-9]|5[0-9])[\-\,\,])*([0-9]|1[0-9]|2[0-9]|3[0-9]|4[0-9]|5[0-9]))|?) .*)'
        # Cron expression validation
      
      storageSize:
        type: string
        pattern: '^\d+(Gi|Ti)$'  # 10Gi, 100Ti, etc.
```

### Controller Validation

What semantic validation happens in the controller during reconciliation?

```python
# Examples:
- If size="xlarge", ensure cluster has enough resources
- If backupSchedule is set, ensure backup destination is reachable
- If replication is enabled, ensure multi-zone cluster
```

---

## 8. Monitoring & Observability

### Metrics

What should operators monitor?

```
database_reconcile_duration_seconds     # How long reconciliation takes
database_ready                          # Is it ready? (0 or 1)
database_size_bytes                     # Current storage size
database_backup_age_seconds             # When was last backup?
database_error_total                    # How many errors?
```

### Events

What important events should be recorded?

```yaml
Normal events:
- DatabaseCreated
- DatabaseReady
- BackupCompleted
- UpgradeStarted
- UpgradeCompleted

Warning/Error events:
- BackupFailed
- LowStorageSpace
- HighErrorRate
- UnhealthyReplica
```

### Logging

What should the controller log?

```
DEBUG: Resource reconciled successfully, status: ready=true
ERROR: Failed to create StatefulSet: [error details]
WARN: PVC creation delay detected, will retry
INFO: Database size upgraded from small to large
```

---

## 9. Testing Strategy

### Manual Testing

How will you test before deployment?

```
1. Create a database with valid spec → Should be ready within 5 minutes
2. Update database size → Should resize without data loss
3. Delete database → Should trigger backup and cleanup
4. Invalid spec → Should reject with clear error message
5. Simultaneous updates → Should handle gracefully
```

### Automated Testing

What automated tests are needed?

- [ ] Unit tests for reconciliation logic
- [ ] Integration tests with a local cluster (kind, minikube)
- [ ] End-to-end tests (full workflow)
- [ ] Performance tests (can it handle 100 databases?)
- [ ] Chaos tests (what if network is down?)

---

## 10. Documentation & Examples

### For Operators (Cluster Admins)

- How to install the operator
- Required permissions
- Configuration options
- Troubleshooting guide

### For Users (Developers)

- Quickstart examples
- Common patterns
- API reference
- Best practices

---

## 11. Timeline & Milestones

When will this be ready?

```
Phase 1: Core functionality     (Week 1-2)
  - CRD definition
  - Basic controller logic
  - Manual testing

Phase 2: Production Ready        (Week 3-4)
  - Error handling
  - Monitoring/logging
  - Documentation

Phase 3: Early Adoption          (Week 5)
  - Limited rollout
  - User feedback
  - Polish

Phase 4: General Availability    (Week 6)
  - Full rollout
  - Support
```

---

## 12. Success Criteria

How will you know when this CRD is successful?

- [ ] Developers can provision databases in < 5 minutes
- [ ] No manual intervention needed for routine operations
- [ ] 99.9% uptime for managed databases
- [ ] Zero data loss in backups
- [ ] Clear, actionable error messages
- [ ] Documentation is complete and tested

---

## Review Checklist

Before implementing, verify:

- [ ] Problem statement is clear
- [ ] Schema is well-designed (not too complex, not too simple)
- [ ] Reconciliation logic handles all scenarios (create, update, delete, error)
- [ ] RBAC permissions are minimal but sufficient
- [ ] Validation is comprehensive (API + controller level)
- [ ] Monitoring/alerting is planned
- [ ] Documentation examples are realistic
- [ ] Timeline is achievable

---

## Feedback & Iteration

Use this document to:

1. **Get feedback from colleagues**: Review architecture before implementing
2. **Identify gaps**: Notice things you hadn't considered
3. **Make decisions**: Resolve design questions early
4. **Communicate intent**: Share vision with your team

---

## Next Steps

Once you've filled out this specification:

1. **Share with stakeholders**: Get approval on design
2. **Create a GitHub issue/PR**: Document the design
3. **Start implementation**: Build the CRD manifest first
4. **Build the controller**: Implement reconciliation logic
5. **Write tests**: Ensure reliability
6. **Deploy**: Release to early adopters
7. **Iterate based on feedback**: Refine based on real usage

---

## Need Help?

If you get stuck on any section:

- **Design questions**: Ask colleagues with Kubernetes experience
- **Technical questions**: Refer back to earlier phases
- **Example patterns**: Look at existing operators on GitHub/OperatorHub
- **Kubernetes docs**: https://kubernetes.io - the official resource

Good luck! 🚀
