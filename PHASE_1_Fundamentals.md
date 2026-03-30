# Phase 1: CRD Fundamentals

## What is a Custom Resource Definition (CRD)?

A **Custom Resource Definition (CRD)** is a Kubernetes API extension that lets you define your own resource types beyond the built-ins like Pods, Deployments, and Services.

**Simple analogy**: If Kubernetes is a database, CRDs are like schema definitions that teach the database about new table types you want to store.

### The Core Idea

By default, Kubernetes understands resources like:
- `Pod` - a containerized application instance
- `Deployment` - a declarative way to manage Pods
- `Service` - a way to expose applications over the network

With a CRD, **you** can define custom resources like:
- `Book` - representing a book in your library system
- `Database` - representing a managed database instance
- `Message` - representing a message queue topic
- `GitRepository` - representing a Git repository to sync

Once you define a CRD, Kubernetes treats your custom resource **exactly like a built-in resource**. You can:
- Store it in etcd (Kubernetes' database)
- Query it with `kubectl get`
- Watch for changes
- Apply YAML manifests to create instances
- Use RBAC to control access

---

## Why CRDs Exist: The Problem They Solve

### Before CRDs (the Hard Way)

Without CRDs, if you wanted Kubernetes to manage a custom resource, you'd need to:
1. Build your own API server from scratch
2. Manage your own database
3. Handle authentication/authorization
4. Implement CRUD operations
5. Make it HTTP-compatible
6. Teach developers how to interact with it
7. Monitor and debug it separately

**This is a massive undertaking.** Most teams didn't have the expertise.

### With CRDs (the Simple Way)

CRDs reuse Kubernetes' existing infrastructure:
- **API Server**: Kubernetes already has one; CRDs plug into it
- **Database (etcd)**: Your custom resources are stored there
- **Authentication/Authorization**: RBAC already works for your resources
- **Validation**: Built-in schema validation
- **Tooling**: `kubectl` already knows how to interact with your resources
- **Monitoring**: Standard Kubernetes patterns work

**Result**: You focus on *business logic*, not infrastructure.

---

## How CRDs Extend the Kubernetes API

### The Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   Kubernetes API Server                     │
│  (single entry point for all resources, built-in & custom) │
└─────────────────────────────────────────────────────────────┘
                           ▲
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
    ┌───────┐          ┌───────┐         ┌───────┐
    │  Pod  │          │  Book │         │ MyApp │
    │ (CRD) │          │ (CRD) │         │ (CRD) │
    └───────┘          └───────┘         └───────┘
    (built-in)         (yours)           (somebody's)

    All stored in etcd, all queryable via API Server, all follow the same rules
```

### What Happens When You Define a CRD

1. You write a CRD manifest (YAML) describing your resource:
   ```yaml
   apiVersion: apiextensions.k8s.io/v1
   kind: CustomResourceDefinition
   metadata:
     name: books.example.com
   spec:
     group: example.com
     scope: Namespaced
     names:
       kind: Book
       plural: books
     versions:
     - name: v1
       served: true
       storage: true
       schema:
         openAPIV3Schema:
           type: object
           properties:
             spec:
               type: object
               properties:
                 title:
                   type: string
   ```

2. You apply it to your cluster: `kubectl apply -f crd.yaml`

3. **Kubernetes registers it with the API server**: The API server now knows about the `Book` kind in the `example.com` API group.

4. **Developers can now use it**:
   ```yaml
   apiVersion: example.com/v1
   kind: Book
   metadata:
     name: my-book
   spec:
     title: "Kubernetes in Action"
   ```

5. **It's stored and queryable like any resource**:
   ```bash
   kubectl get books          # List all books
   kubectl describe book my-book  # Show details
   kubectl delete book my-book    # Delete it
   ```

---

## Key Terminology: API Groups, Versions, and Kinds

You need to understand three concepts to work with CRDs:

### 1. **API Group**

The **namespace** for your resources, using reverse-domain notation.

**Examples**:
- `core` (or empty) - built-in Kubernetes resources (Pods, Services)
- `apps` - built-in resources (Deployments, StatefulSets)
- `example.com` - your organization's custom resources
- `databases.acme.io` - database operator resources

**Purpose**: Avoid naming conflicts. If I create a `Book` CRD in `api.library.io`, and you create a `Book` CRD in `api.bookstore.io`, they don't conflict—they're different resources.

### 2. **Kind**

The **resource type** name (singular, PascalCase).

**Examples**:
- `Pod`, `Deployment`, `Service` (built-in)
- `Book`, `Database`, `GitRepository` (custom)

**Usage**: In YAML, `kind: Book` tells Kubernetes you're creating a Book resource.

### 3. **Version**

The **API version** for your resource. Allows you to evolve your CRD over time.

**Examples**:
- `v1` - stable version
- `v1beta1` - beta version (might change)
- `v2` - major version change

**Format**: `<apiGroup>/<version>`
- `core/v1` or just `v1` (built-in core group)
- `apps/v1` (apps group)
- `example.com/v1` (your custom group)
- `example.com/v1beta1` (beta version)

**Usage in YAML**:
```yaml
apiVersion: example.com/v1
kind: Book
metadata:
  name: my-book
```

**Diagram**:
```
apiVersion: example.com/v1
            ├─ example.com (API Group)
            └─ v1 (Version)

kind: Book
     └─ Book (Kind)
```

---

## CRD vs. Custom Resource (Instance): The Distinction

### CRD = Recipe, Custom Resource = Actual Item

| Aspect | CRD | Custom Resource |
|--------|-----|-----------------|
| **What is it?** | A *definition* or *schema* | An *instance* or *object* |
| **Analogy** | A cookie recipe | An actual cookie |
| **Example** | Defines what a `Book` can be | An actual book named `my-book` |
| **In YAML** | `kind: CustomResourceDefinition` | `kind: Book` |
| **Created by** | Cluster admin (once) | Developers/apps (many times) |
| **kubectl command** | Usually not queried; implicit | `kubectl get books` |

### Example: The Book CRD

**Step 1: Define the CRD (once)**
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: books.example.com
spec:
  group: example.com
  names:
    kind: Book
    plural: books
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              title:
                type: string
                description: "Title of the book"
              author:
                type: string
                description: "Author of the book"
```

Now Kubernetes knows what a `Book` is.

**Step 2: Create instances of the custom resource (many times)**
```yaml
apiVersion: example.com/v1
kind: Book
metadata:
  name: kubernetes-in-action
  namespace: default
spec:
  title: "Kubernetes in Action"
  author: "Marko Luksa"
---
apiVersion: example.com/v1
kind: Book
metadata:
  name: the-phoenix-project
  namespace: default
spec:
  title: "The Phoenix Project"
  author: "Gene Kim"
```

Two instances of the `Book` custom resource. Each one is stored separately in etcd.

---

## The Kubernetes Architecture Picture

### How Everything Fits Together

```
┌─────────────────────────────────────────────────────────┐
│            Kubernetes Cluster                           │
│                                                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │         API Server (Single Entry Point)          │  │
│  │  - Validates requests                            │  │
│  │  - Stores resources in etcd                      │  │
│  │  - Watches for changes                           │  │
│  │  - Enforces RBAC                                 │  │
│  └──────────────────────────────────────────────────┘  │
│                      ▲                                  │
│   ┌──────────────────┼──────────────────┐              │
│   │                  │                  │              │
│ ┌─────┐          ┌───────┐          ┌─────────┐       │
│ │ Pod │          │ Book  │          │ MyApp   │       │
│ └─────┘          └───────┘          └─────────┘       │
│(built-in)      (defined by           (yours)          │
│ resource      your new CRD)       custom CRD           │
│                                                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │  etcd (Persistent Storage)                       │  │
│  │  - Stores all resources (built-in & custom)     │  │
│  │  - Each object has same structure fundamentally  │  │
│  └──────────────────────────────────────────────────┘  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Common Questions Beginners Ask

### Q1: "Is my CRD 'real' Kubernetes?"

**A**: Yes! Once defined, your custom resources are treated identically to built-in resources by the API server. The only difference is that built-in resources have controllers that manage them; custom resources typically have controllers *you* write.

### Q2: "Who uses my CRD?"

**A**: 
- **Cluster admins**: Install the CRD (one-time)
- **Developers**: Create instances of the custom resource
- **Controllers**: Watch for changes and react (this is where your business logic goes)

### Q3: "Do I need a controller for my CRD?"

**A**: Not necessarily. You can define a CRD and manually manage instances (though that defeats much of the purpose). Typically, a **controller** watches your CRD instances and implements the business logic that makes them useful.

### Q4: "What's the difference between a CRD and an Operator?"

**A**: 
- **CRD** = The resource definition (like a Pod schema)
- **Operator** = CRD + Controller(s) + Operational Knowledge packaged together

An Operator is usually deployed as a container that runs in your cluster, watches your CRD instances, and takes actions based on them.

---

## What You Now Understand

✅ CRDs extend Kubernetes to define custom resource types  
✅ CRDs leverage Kubernetes' existing API infrastructure  
✅ API Group + Version + Kind uniquely identify a resource  
✅ A CRD is a definition; a Custom Resource is an instance  
✅ CRDs typically work with controllers to provide behavior  

---

## Next: Phase 2

In Phase 2, you'll write your first CRD—a `Book` CRD with validation rules. You'll learn the YAML structure and test it hands-on.

**Ready?** Proceed to `PHASE_2_YAML_and_Validation.md`
