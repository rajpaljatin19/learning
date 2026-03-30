# Phase 2: YAML Specification & Schema Validation

Now that you understand *what* CRDs are, let's learn *how to define them*.

---

## CRD Manifest Structure

A CRD manifest is a Kubernetes YAML file that describes your custom resource. Here's the anatomy:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: books.example.com                          # Must be <plural>.<group>
spec:
  group: example.com                               # API group (reverse-domain format)
  scope: Namespaced                                # Namespaced or Cluster
  names:
    kind: Book                                     # Singular, PascalCase
    plural: books                                  # Lowercase plural
    shortNames:                                    # Optional: short command-line names
    - bk
  versions:
  - name: v1
    served: true                                   # Can clients use this version?
    storage: true                                  # Where is data stored? (only one can be true)
    schema:
      openAPIV3Schema:                             # Validation schema
        type: object
        description: "A Book resource"
        properties:
          # ... define fields here (see below)
```

### Breaking It Down

| Field | Purpose |
|-------|---------|
| `apiVersion: apiextensions.k8s.io/v1` | Tells Kubernetes: "This is a CRD definition" (part of the API extensions group) |
| `kind: CustomResourceDefinition` | This is a CRD manifest |
| `metadata.name` | Name of the CRD itself. Format: `<plural>.<group>`. Example: `books.example.com` |
| `spec.group` | Your API group (reverse-domain format). Example: `example.com`, `databases.acme.io` |
| `spec.scope` | `Namespaced` (lives in a namespace) or `Cluster` (cluster-wide). Most are Namespaced. |
| `spec.names.kind` | The resource type name (singular, PascalCase). Users write `kind: Book` in their YAML. |
| `spec.names.plural` | Plural form for CLI commands. Users type `kubectl get books`. |
| `spec.names.shortNames` | Optional aliases. Users can type `kubectl get bk` instead of `kubectl get books`. |
| `spec.versions[].name` | Version label (usually `v1`, `v1beta1`, etc.). |
| `spec.versions[].served` | Can clients currently use this version? (For gradual API deprecation) |
| `spec.versions[].storage` | Which version is the "source of truth" in etcd? Only one version can have this as `true`. |
| `spec.versions[].schema.openAPIV3Schema` | The validation schema (what fields are allowed, types, constraints, etc.) |

---

## OpenAPI 3.0 Schema: Defining Fields

The `openAPIV3Schema` is where you define what fields your custom resource can have and what constraints they must satisfy.

### Simple Example: A Book CRD

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
        description: "A Book resource"
        properties:
          spec:
            type: object
            description: "Book specification (desired state)"
            properties:
              title:
                type: string
                description: "Title of the book"
              author:
                type: string
                description: "Author of the book"
              pages:
                type: integer
                description: "Number of pages"
              isbn:
                type: string
                description: "ISBN-13"
```

**What does this mean?**

When a user creates a Book instance like:
```yaml
apiVersion: example.com/v1
kind: Book
metadata:
  name: my-book
spec:
  title: "Kubernetes in Action"
  author: "Marko Luksa"
  pages: 500
  isbn: "1234567890123"
```

Kubernetes validates:
- `spec.title` must be a string ✓
- `spec.author` must be a string ✓
- `spec.pages` must be an integer ✓
- `spec.isbn` must be a string ✓

### Common Schema Field Types

```yaml
properties:
  # String types
  name:
    type: string

  # Integer types
  count:
    type: integer

  # Boolean types
  enabled:
    type: boolean

  # Floating-point numbers
  rating:
    type: number

  # Arrays
  tags:
    type: array
    items:
      type: string
    # Example value: ["fiction", "adventure", "sci-fi"]

  # Objects (nested structures)
  author:
    type: object
    properties:
      name:
        type: string
      email:
        type: string

  # Enums (only specific values allowed)
  status:
    type: string
    enum: ["draft", "published", "out-of-print"]
```

---

## Validation Rules: Making Your Schema Strict

### 1. Required Fields

By default, all fields are optional. To make fields required:

```yaml
spec:
  type: object
  properties:
    title:
      type: string
    author:
      type: string
  required:                     # These fields must be present
  - title
  - author
```

**Effect**: When someone creates a Book without a `title` or `author`, Kubernetes rejects it with a validation error.

### 2. Field Constraints

#### Minimum/Maximum Values (for integers)

```yaml
pages:
  type: integer
  minimum: 1                    # Must be 1 or greater
  maximum: 10000                # Must be 10000 or less
```

#### String Length

```yaml
title:
  type: string
  minLength: 1
  maxLength: 256
```

#### Pattern Matching (Regex)

```yaml
isbn:
  type: string
  pattern: '^\d{13}$'           # Must be exactly 13 digits
  description: "ISBN-13 format"
```

#### Enum (Specific Allowed Values)

```yaml
status:
  type: string
  enum:
  - draft
  - published
  - archived
```

### 3. Defaults

You can provide default values:

```yaml
status:
  type: string
  enum: ["draft", "published", "archived"]
  default: "draft"
```

When a user creates a Book without specifying `status`, Kubernetes automatically sets it to `"draft"`.

---

## Complete Book CRD Example with Validation

Here's a realistic, fully-specified Book CRD:

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
    shortNames:
    - bk
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        description: "A Book in our library system"
        required:
        - spec
        properties:
          apiVersion:
            type: string
            description: "API version (handled by Kubernetes)"
          kind:
            type: string
            description: "Kind (handled by Kubernetes)"
          metadata:
            type: object
            description: "Object metadata (name, namespace, labels, etc.)"
          spec:
            type: object
            description: "Book specification (desired state)"
            required:
            - title
            - author
            properties:
              title:
                type: string
                minLength: 1
                maxLength: 256
                description: "Title of the book"
              author:
                type: string
                minLength: 1
                description: "Author of the book"
              pages:
                type: integer
                minimum: 1
                maximum: 100000
                description: "Number of pages"
              isbn:
                type: string
                pattern: '^\d{13}$'
                description: "ISBN-13 (13 digits)"
              publicationYear:
                type: integer
                minimum: 1000
                maximum: 2100
                description: "Year of publication"
              genre:
                type: string
                enum:
                - fiction
                - non-fiction
                - mystery
                - science_fiction
                - romance
                - biography
                description: "Genre of the book"
              available:
                type: boolean
                default: true
                description: "Is the book currently available?"
              tags:
                type: array
                items:
                  type: string
                maxItems: 10
                description: "Tags describing the book (max 10)"
          status:
            type: object
            description: "Book status (actual state, managed by controller)"
            properties:
              borrowedBy:
                type: string
                description: "Name of person who borrowed the book"
              lastBorrowedDate:
                type: string
                format: date-time
                description: "When was it last borrowed"
              checkouts:
                type: integer
                description: "Total number of times borrowed"
```

### What This CRD Enforces

✅ `title` is required, max 256 characters  
✅ `author` is required  
✅ `pages` must be between 1 and 100,000  
✅ `isbn` must be exactly 13 digits  
✅ `genre` can only be one of the allowed values  
✅ `available` defaults to `true`  
✅ `tags` can have at most 10 items  
✅ `status` (actual state) is separate from `spec` (desired state)  

---

## Spec vs. Status: Desired vs. Actual State

This is a crucial Kubernetes pattern:

### `spec`: What the User Wants (Desired State)

The user specifies what they want the resource to be like. Example:
```yaml
spec:
  title: "Kubernetes in Action"
  author: "Marko Luksa"
  available: true
```

The user says: "I want this book to be available."

### `status`: What Actually Is (Actual State)

The controller updates this field to reflect reality. Example:
```yaml
status:
  borrowedBy: "Alice"
  lastBorrowedDate: "2025-03-15T10:30:00Z"
  checkouts: 5
```

The controller says: "Actually, the book is borrowed by Alice."

### The Feedback Loop

```
User writes spec (desired):
"I want this available"

Manager/Controller reads it:
"The user wants this available. Let me check..."

Controller updates status (actual):
"This book is borrowed by Alice (not available)"

User sees status and adjusts spec if needed
```

**Important**: 
- Users edit `spec` directly
- Controllers edit `status` automatically
- Kubernetes keeps them separate to avoid conflicts

---

## Status Subresource

To properly implement spec vs. status, use the `status` subresource:

```yaml
versions:
- name: v1
  served: true
  storage: true
  subresources:
    status: {}                  # Enable the status subresource
  schema:
    openAPIV3Schema:
      # ... your schema here
```

With this enabled:
- `kubectl apply` updates only `spec`
- Controllers use a special API call to update `status`
- No accidental conflicts between the two

---

## Hands-On Exercise: Your First CRD

### Exercise 1: Create the Book CRD

Create a file called `book-crd.yaml`:

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
    shortNames:
    - bk
  versions:
  - name: v1
    served: true
    storage: true
    subresources:
      status: {}
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required:
            - title
            - author
            properties:
              title:
                type: string
                minLength: 1
                maxLength: 256
              author:
                type: string
                minLength: 1
              pages:
                type: integer
                minimum: 1
              genre:
                type: string
                enum:
                - fiction
                - non-fiction
                - mystery
                - science_fiction
          status:
            type: object
            properties:
              borrowedBy:
                type: string
              checkouts:
                type: integer
```

### Exercise 2: Apply It to a Local Cluster

Assuming you have a local Kubernetes cluster running (minikube, kind, Docker Desktop, etc.):

```bash
kubectl apply -f book-crd.yaml
```

**Check it was created:**
```bash
kubectl get crd books.example.com
```

### Exercise 3: Create Valid Book Instances

Create a file called `books.yaml`:

```yaml
apiVersion: example.com/v1
kind: Book
metadata:
  name: k8s-in-action
spec:
  title: "Kubernetes in Action"
  author: "Marko Luksa"
  pages: 500
  genre: "non-fiction"
---
apiVersion: example.com/v1
kind: Book
metadata:
  name: the-phoenix-project
spec:
  title: "The Phoenix Project"
  author: "Gene Kim"
  genre: "non-fiction"
```

Apply it:
```bash
kubectl apply -f books.yaml
```

List your books:
```bash
kubectl get books
kubectl get bk                    # Using short name
```

Describe a book:
```bash
kubectl describe book k8s-in-action
```

### Exercise 4: Test Validation (Create an Invalid Instance)

Try creating an invalid book (missing required `author`):

```yaml
apiVersion: example.com/v1
kind: Book
metadata:
  name: invalid-book
spec:
  title: "A Book"
  # author is missing!
  genre: "fiction"
```

Save it and try:
```bash
kubectl apply -f invalid-book.yaml
```

**Expected result**: Kubernetes rejects it with an error saying `author` is required. ✓

### Exercise 5: Test Field Constraints

Try creating a book with an invalid genre (not in the enum):

```yaml
apiVersion: example.com/v1
kind: Book
metadata:
  name: weird-book
spec:
  title: "Weird"
  author: "Someone"
  genre: "horror"                # Not in allowed list!
```

**Expected result**: Kubernetes rejects it with an error about `genre`. ✓

### Exercise 6: Update Status

Simulate a controller setting the status (you do it manually):

```bash
kubectl patch book k8s-in-action --type merge -p '{"status":{"borrowedBy":"Alice","checkouts":3}}'
```

Check it worked:
```bash
kubectl get book k8s-in-action -o yaml
```

You'll see the `status` is updated without affecting `spec`. ✓

---

## Key Takeaways

✅ CRD manifests define API groups, scopes, names, and validation  
✅ OpenAPI 3.0 schemas validate field types and constraints  
✅ Required fields, min/max values, enums, and patterns enforce data integrity  
✅ Spec (desired state) and status (actual state) are separate  
✅ You can create and manage custom resources like any Kubernetes resource  

---

## Next: Phase 3

In Phase 3, you'll learn about **controllers**—the piece that reads your CRD instances and makes them actually *do* something.

**Ready?** Proceed to `PHASE_3_Controllers_and_Reconciliation.md`
