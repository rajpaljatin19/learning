# Phase 3: Controllers & Reconciliation

Now you know how to *define* a CRD. But what makes it actually *useful*?

**Answer**: Controllers.

A controller is a program running in your cluster that watches your custom resources and takes action when they change. This is where your business logic lives.

---

## What is a Controller?

### Simple Definition

A **controller** is an **infinite loop** that:
1. Watches for changes to your resources (via the API server)
2. Reads the current state
3. Compares it against desired state (spec)
4. Takes action to match them
5. Updates the status
6. Repeats

### Analogy: Thermostat

A thermostat is like a controller:

```
┌─────────────────────────────────────────────────┐
│ Thermostat (Controller)                         │
│                                                 │
│ while true:                                     │
│   - Read desired temp (spec)       "We want 72F"
│   - Read actual temp (status)      "It's 68F"
│   - If actual != desired:                       
│     - Turn on heat (take action)   "Turn on heat"
│   - Update status                  "Heat is on"
│   - Wait and loop again                         
└─────────────────────────────────────────────────┘
```

Controllers work the same way with Kubernetes resources.

---

## The Reconciliation Loop

The core concept of any controller is **reconciliation**:

### What is Reconciliation?

**Reconciliation** = Making actual state match desired state.

```
Desired State (spec)              Actual State (status)
─────────────────────             ──────────────────────
Book: "K8s in Action"      →→→    Borrowed by: Alice
Availability: true               Not available
Want this                         Currently this

Controller's job: "That doesn't match! Let me fix it."
```

### The Reconciliation Loop Flowchart

```
┌──────────────────────────────────────────────────┐
│ Start (Watch for resource changes)               │
└──────────────────────────────────────────────────┘
  │
  ├─→ Event: Book created/updated/deleted
  │
  ├─→ "Read the Book spec and current status"
  │
  ├─→ "Does actual state match desired state?"
  │   │
  │   ├─ Yes? Do nothing, loop again
  │   │
  │   └─ No? Take action to fix it
  │
  ├─→ "Update the status subresource"
  │
  └─→ Back to watching...
```

---

## Example: A Book Availability Controller

Let's say you want a controller that:
- Watches `Book` resources
- If a book is marked as `available: true` in the spec but someone borrowed it, updates status to show it's borrowed
- If a book should be `available` but hasn't been touched in 30 days, marks it as returned

### Step 1: Define the CRD (We Did This)

```yaml
apiVersion: example.com/v1
kind: Book
metadata:
  name: my-book
spec:
  title: "Kubernetes in Action"
  author: "Marko Luksa"
  available: true                    # Desired state
status:
  borrowedBy: ""
  lastBorrowedDate: null             # Actual state
```

### Step 2: Write the Controller Logic (Pseudocode)

```python
def reconcile_book(book):
    """
    Reconcile a Book resource.
    Make actual state (status) match desired state (spec).
    """
    
    # Read desired state
    desired_available = book.spec.available
    
    # Read actual state
    actual_borrowed_by = book.status.borrowedBy
    actual_is_borrowed = actual_borrowed_by != ""
    
    # Check if they match
    if desired_available and actual_is_borrowed:
        # Mismatch! User wants it available, but it's borrowed.
        # Take action: Notify someone? Auto-recall it?
        # For now, update status to note the mismatch
        book.status.issue = "Book marked as available but is borrowed"
    
    elif not desired_available and not actual_is_borrowed:
        # Mismatch! User wants it unavailable, but it's not borrowed.
        # Take action: Maybe mark it as under maintenance?
        book.status.issue = "Book marked as unavailable but not currently borrowed"
    
    else:
        # States match, great!
        book.status.issue = ""
    
    # Store updated status back to API server
    update_book_status(book)
```

### Step 3: Run the Controller as a Loop

In a real controller (written in Go, Python, etc.), you'd do something like:

```python
def watch_books():
    """Continuously watch for Book changes and reconcile."""
    
    while True:
        # Watch for events (created, updated, deleted)
        for event in api_server.watch(resource_type="Book"):
            book = event.object
            
            # Reconcile the book
            reconcile_book(book)
            
            # If there's an error, log it and retry later
            # (Don't crash the controller)
```

---

## How Controllers Interact with the API Server

### The Watch Mechanism

```
┌──────────────────────────────┐
│ Kubernetes API Server        │
│ (stores Books in etcd)       │
└──────────────────────────────┘
     ▲                    │
     │                    │
  UPDATE               WATCH (stream of events)
  status                 "Book created!"
     │                    "Book updated!"
     │                    "Book deleted!"
     │                    .....
     │                    │
     └────────────────────┴───────────────────┐
                                              │
                            ┌─────────────────┴──┐
                            │ Book Controller    │
                            │ (Running in Pod)   │
                            │                    │
                            │ while true:        │
                            │   watch events     │
                            │   reconcile()      │
                            │   update status    │
                            └────────────────────┘
```

### The Flow

1. **User creates a Book**: 
   ```bash
   kubectl apply -f book.yaml
   ```

2. **API Server stores it**:
   - Validates against the CRD schema ✓
   - Saves to etcd ✓
   
3. **API Server sends event to controller**:
   - "A Book named 'my-book' was created in namespace 'default'"
   
4. **Controller receives event & reads the Book**:
   - Fetches full Book object from API server
   - Examines spec and current status
   
5. **Controller reconciles**:
   - Compares desired state (spec) to actual state (status)
   - Takes action if they don't match
   
6. **Controller updates status**:
   - Makes an API call to update the status subresource
   - This does NOT update spec (they're separate)
   
7. **API Server updates etcd**:
   - Status is now stored
   
8. **User sees the result**:
   - `kubectl get book my-book -o yaml` shows updated status

---

## Reconciliation: Deep Dive

### The Idempotency Principle

**Critical rule**: Reconciliation must be **idempotent**.

**What does idempotent mean?**

If you run the same reconciliation 10 times without any changes, you get the same result.

**Why?** Because Kubernetes will call your reconciliation repeatedly:
- Every time a resource changes
- Every few seconds (as a background check)
- On controller restart
- On transient errors

**Bad (non-idempotent) logic**:
```python
def reconcile(book):
    book.status.checkouts += 1  # ❌ WRONG! Increments every time
    update_status(book)
```

If Kubernetes calls this 5 times, checkouts goes up by 5. That's a bug.

**Good (idempotent) logic**:
```python
def reconcile(book):
    # Check the current state
    borrowed = book.status.borrowedBy != ""
    
    # Take action only if needed
    if borrowed and book.spec.available:
        book.status.issue = "Book marked as available but borrowed"
    else:
        book.status.issue = ""
    
    update_status(book)  # ✓ CORRECT! Same result every time
```

No matter how many times Kubernetes calls this, the result is the same.

---

## Owner References & Garbage Collection

Often, a controller creates *other* resources as a side effect.

### Example

Imagine a `Database` CRD. When someone creates a Database instance, the controller:
1. Reads the Database spec (desired database configuration)
2. Creates PersistentVolumes, Secrets, ConfigMaps to support it
3. Updates the Database status

But what if the user deletes the Database? Should the PersistentVolumes also be deleted?

**Yes!** Using **owner references**.

### How It Works

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: db-pv-001
  ownerReferences:
  - apiVersion: example.com/v1
    kind: Database
    name: my-database        # ← Points back to the Database resource
    uid: abc123
```

When the Database is deleted, Kubernetes automatically deletes all resources that reference it.

---

## Status Conditions: Communicating Complex State

Real-world resources often have complex states. Use **conditions**:

```yaml
status:
  conditions:
  - type: Ready
    status: "True"
    lastTransitionTime: "2025-03-15T10:30:00Z"
    reason: "ResourceReady"
    message: "Book is ready for checkout"
  - type: Available
    status: "False"
    lastTransitionTime: "2025-03-15T10:45:00Z"
    reason: "Borrowed"
    message: "Currently borrowed by Alice"
```

This allows complex, multi-faceted status without confusion.

---

## Hands-On Exercise: Manual Reconciliation

You don't have a real controller yet, but you can simulate one with `kubectl`.

### Setup: Create Books

```bash
# Apply the CRD
kubectl apply -f book-crd.yaml

# Create a book
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: Book
metadata:
  name: test-book
spec:
  title: "Test Book"
  author: "Test Author"
  available: true
EOF
```

### Simulate a Controller: Check & Reconcile

**Step 1: Read the desired state**
```bash
kubectl get book test-book -o yaml
```

Look at `spec.available`. Let's say it's `true`.

**Step 2: Simulate an event (someone borrows the book)**

Pretend someone borrowed the book. Update status:
```bash
kubectl patch book test-book --type merge \
  -p '{"status":{"borrowedBy":"Alice"}}'
```

Check it:
```bash
kubectl get book test-book -o yaml
```

Now you see:
- `spec.available: true` (desired: book should be available)
- `status.borrowedBy: "Alice"` (actual: it's borrowed)

**Mismatch!**

**Step 3: Reconcile (take action)**

The controller should detect this mismatch. Simulate it:

```bash
kubectl patch book test-book --type merge \
  -p '{"status":{"issue":"Mismatch: marked available but borrowed"}}'
```

Or, the controller could:
- Send a notification
- Auto-recall the book
- Mark it for review

You've just simulated what a real controller does!

### Exercise: More Complex Scenario

1. Create a book with `available: false`
2. Manually update status to `borrowedBy: ""`
3. A controller should notice this mismatch and either:
   - Return the book automatically
   - Mark `available: true`
   - Raise an alert

This is exactly what production controllers do!

---

## Retry Logic & Errors

Real controllers handle errors gracefully:

```python
def watch_and_reconcile():
    while True:
        for event in watch_events():
            try:
                reconcile(event.object)
            except API_Error as e:
                # Don't crash. Log and retry later.
                logger.error(f"Failed to reconcile: {e}")
                # Kubernetes will send the event again
            except Exception as e:
                # Unknown error. Still log and continue.
                logger.error(f"Unexpected error: {e}")
```

Kubernetes' watch mechanism automatically re-queues failed reconciliations, so you don't have to worry about missing updates.

---

## Watch & Event-Driven Architecture

### Why Watch?

Instead of polling every resource every 5 seconds (wasteful), Kubernetes uses a **watch** mechanism:

```
┌─────────────────────────┐
│ Kubernetes API Server   │
│                         │
│ INEFFICIENT (Polling):  │
│ - Ask every 5 seconds   │
│ - Wasted calls          │
│ - Delayed detection     │
│                         │
│ EFFICIENT (Watching):   │
│ - API sends events      │
│ - Instant notification  │
│ - Lower overhead        │
└─────────────────────────┘
```

### Watch is a Stream

```bash
kubectl get books --watch
```

This opens a persistent connection and streams events:

```
NAME          READY   STATUS    RESTARTS   AGE
test-book     0/1     Added     0          1s
test-book     0/1     Updated   0          2s
another-book  1/1     Added     0          3s
test-book     0/1     Deleted   0          5s
```

Your controller does this automatically, but for all your custom resources.

---

## Key Takeaways

✅ Controllers watch resources and implement business logic  
✅ Reconciliation loop: watch → compare → act → update status  
✅ Reconciliation must be idempotent (same result no matter how many times)  
✅ Controllers can create/manage other Kubernetes resources  
✅ Owner references handle cascade deletion  
✅ Status conditions communicate complex state  
✅ Watch mechanism is efficient and event-driven  

---

## Next: Phase 4

In Phase 4, you'll learn how controllers are **packaged and deployed** as Operators, and discover best practices for production CRDs.

**Ready?** Proceed to `PHASE_4_Operators_and_Best_Practices.md`
