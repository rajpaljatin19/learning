# Kubernetes CRD Learning Plan - Complete Guide

Welcome! This folder contains everything you need to understand Kubernetes Custom Resource Definitions (CRDs) from first principles.

---

## 📚 How to Use This Guide

### Recommended Reading Order

1. **Start here**: [README_START_HERE.md](README_START_HERE.md)
2. **Phase 1**: [PHASE_1_Fundamentals.md](PHASE_1_Fundamentals.md) - Understand *what* CRDs are
3. **Phase 2**: [PHASE_2_YAML_and_Validation.md](PHASE_2_YAML_and_Validation.md) - Learn *how* to define them
4. **Phase 3**: [PHASE_3_Controllers_and_Reconciliation.md](PHASE_3_Controllers_and_Reconciliation.md) - Learn *how they work*
5. **Phase 4**: [PHASE_4_Operators_and_Best_Practices.md](PHASE_4_Operators_and_Best_Practices.md) - Learn production patterns
6. **Phase 5**: [PHASE_5_Implementation_Planning.md](PHASE_5_Implementation_Planning.md) - Plan *your* CRD
7. **Reference**: [QUICK_REFERENCE.md](QUICK_REFERENCE.md) - One-page lookups

### Time Commitment

- **Full learning plan**: 3-5 hours of reading + hands-on
- **Each phase**: 30-60 minutes
- **Hands-on exercises**: 20-30 minutes per phase

### Prerequisites

- Basic Kubernetes knowledge (Pods, Deployments, Services)
- A local Kubernetes cluster (Docker Desktop, minikube, or kind)
- `kubectl` installed
- Basic familiarity with YAML
- Text editor or IDE

---

## 📖 What's In This Folder

### Core Learning Materials

| File | Purpose | Time |
|------|---------|------|
| `PHASE_1_Fundamentals.md` | Conceptual foundation | 40 min |
| `PHASE_2_YAML_and_Validation.md` | Hands-on CRD definition | 50 min |
| `PHASE_3_Controllers_and_Reconciliation.md` | How resources are managed | 50 min |
| `PHASE_4_Operators_and_Best_Practices.md` | Production patterns | 45 min |
| `PHASE_5_Implementation_Planning.md` | Design your CRD | 30 min |

### Reference Materials

| File | Purpose |
|------|---------|
| `QUICK_REFERENCE.md` | One-page syntax/concept reference |
| `README_START_HERE.md` | Quick overview + learning tips |

### Example Files (In `/examples/` if included)

- `book-crd.yaml` - Complete Book CRD example
- `book-instances.yaml` - Example Book resources
- `controller-rbac.yaml` - RBAC template for controllers

---

## 🎯 What You'll Learn

### Phase 1: Fundamentals
✅ What is a Custom Resource Definition (CRD)?  
✅ Why CRDs exist and the problem they solve  
✅ How CRDs extend the Kubernetes API server  
✅ Key terminology: API groups, versions, kinds  
✅ Relationship between CRDs (definition) and Custom Resources (instances)  

**Success metric**: You can explain CRDs to a colleague in 2 minutes.

### Phase 2: YAML & Validation
✅ CRD manifest structure (spec, schema, versions)  
✅ OpenAPI 3.0 schema for validation  
✅ Field types and constraints (min/max, pattern, enum)  
✅ Required fields and defaults  
✅ Spec vs. Status subresource  

**Success metric**: You can write a CRD with validation rules and test it on a local cluster.

### Phase 3: Controllers & Reconciliation
✅ What controllers are and why they're needed  
✅ The reconciliation loop (watch → compare → act → update)  
✅ Idempotency and why it matters  
✅ Watch mechanism and event-driven architecture  
✅ Owner references and garbage collection  

**Success metric**: You understand how controllers drive behavior and can manually simulate reconciliation.

### Phase 4: Operators & Best Practices
✅ What an Operator is (CRD + Controller + Knowledge)  
✅ RBAC and controller permissions  
✅ Single responsibility principle  
✅ Error handling and status conditions  
✅ Real-world operator patterns  

**Success metric**: You can identify best practices in existing operators and apply them to your design.

### Phase 5: Implementation Planning
✅ Document your CRD use case  
✅ Design your resource schema  
✅ Plan reconciliation logic  
✅ Identify dependencies and error scenarios  
✅ Create a specification document for implementation  

**Success metric**: You have a clear, written specification ready for implementation.

---

## 🚀 Hands-On Exercises

Each phase includes hands-on exercises. You'll:

1. **Phase 2**: Create a Book CRD, create instances, test validation
2. **Phase 3**: Manually update status using kubectl (simulate controller)
3. **Phase 4**: Design RBAC for a hypothetical controller
4. **Phase 5**: Fill out specification template for your CRD

**Time**: 20-30 minutes per phase (optional but recommended)

---

## 💡 Learning Tips

### Concept Mapping

Try to internalize these relationships:

```
CRD (Definition)           Custom Resource (Instance)
      ↓                           ↓
Schema Definition          Actual Data
      ↓                           ↓
Like a database table      Like a row in the table
      ↓                           ↓
Pod Kind definition        An actual Pod
```

### Draw Diagrams

As you learn, sketch:
- API server architecture
- Controller reconciliation loop
- Operator packaging structure

Visual learning helps! Even stick figures are fine.

### Experiment Locally

Each phase has a "hands-on" section. **Do them**. Real experience beats only reading.

### Ask Questions

If something doesn't make sense:
1. Re-read the relevant section
2. Check the Quick Reference
3. Look for analogies in the text
4. Try it hands-on to understand

### Use the Specification Template

Phase 5's specification template isn't just for planning. Use it to:
- Organize your thoughts
- Document decisions
- Share with colleagues for feedback
- Refer back when implementation gets confusing

---

## 🔗 External Resources

### Official Kubernetes Docs
- [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
- [CustomResourceDefinitions](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)
- [API Conventions](https://kubernetes.io/docs/concepts/overview/kubernetes-api/)

### Learning Tools
- [Operator SDK](https://sdk.operatorframework.io/) - Framework for building operators
- [OperatorHub.io](https://operatorhub.io) - Marketplace for discovering operators
- [Kind](https://kind.sigs.k8s.io/) - Local Kubernetes cluster
- [Minikube](https://minikube.sigs.k8s.io/) - Local Kubernetes cluster

### Example Operators to Study
- [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator)
- [Cert-Manager](https://github.com/cert-manager/cert-manager)
- [ArgoCD](https://github.com/argoproj/argo-cd)
- [Redis Operator](https://github.com/spotahome/redis-operator)

---

## ❓ FAQ

### Q: Do I need a real Kubernetes cluster?
**A**: No, a local cluster (Docker Desktop, minikube, kind) works great for learning. You can follow 80% of the exercises without a real cluster.

### Q: How deep should I go?
**A**: Read all 5 phases. Do the hands-on exercises. Refer back when building your operator.

### Q: What if I get stuck on a concept?
**A**: 
1. Re-read that section
2. Check Quick Reference
3. Try the hands-on exercise
4. Draw a diagram
5. Sleep on it, come back fresh

### Q: Should I learn to code a controller now?
**A**: No! This plan focuses on understanding *concepts*. Controller code (Go, Python, etc.) is a separate skill. Learn architecture first, then implementation details.

### Q: Can I skip phases?
**A**: Not recommended. Each phase builds on the previous. Phase 1 is essential; the others can be adapted based on your background.

### Q: What's the difference between a CRD and an API?
**A**: A CRD is a specific Kubernetes extension mechanism. CRDs leverage the existing Kubernetes API. Not all custom APIs are CRDs (some are external services).

---

## 📊 Learning Checklist

Use this to track your progress:

- [ ] **Phase 1 Complete**
  - [ ] I understand what CRDs are
  - [ ] I understand why Kubernetes has CRDs
  - [ ] I know the relationship between CRD (definition) and Custom Resource (instance)
  - [ ] I can explain API groups, kinds, and versions

- [ ] **Phase 2 Complete**
  - [ ] I can write a CRD manifest
  - [ ] I understand OpenAPI 3.0 schema
  - [ ] I've created a working CRD on my cluster
  - [ ] I've created instances and tested validation

- [ ] **Phase 3 Complete**
  - [ ] I understand the reconciliation loop
  - [ ] I know what idempotency means and why it matters
  - [ ] I've manually simulated controller behavior
  - [ ] I understand spec vs. status

- [ ] **Phase 4 Complete**
  - [ ] I understand what an Operator is
  - [ ] I can design RBAC for a controller
  - [ ] I know best practices for CRD design
  - [ ] I can identify patterns in real operators

- [ ] **Phase 5 Complete**
  - [ ] I've filled out the specification template for my CRD
  - [ ] I've designed my schema
  - [ ] I've planned my reconciliation logic
  - [ ] I've identified dependencies and error scenarios

- [ ] **Ready to Implement**
  - [ ] I have a clear specification
  - [ ] I understand what the controller needs to do
  - [ ] I can sketch the architecture
  - [ ] I'm ready to write code

---

## 🎓 After This Plan

Once you've completed all 5 phases:

1. **You're ready to build**: Start implementing your CRD and controller
2. **Reference as needed**: Come back to Quick Reference while coding
3. **Study real operators**: Read open-source operator code to see patterns
4. **Iterate**: Build simple → complex, test thoroughly, gather feedback
5. **Share**: Publish your operator and help others learn

---

## 📝 Quick Navigation

### By Concept
- **Want to understand basics?** → Phase 1
- **Want to write YAML?** → Phase 2
- **Want to understand reconciliation?** → Phase 3
- **Want to know best practices?** → Phase 4
- **Want to plan your CRD?** → Phase 5
- **Need a syntax reference?** → Quick Reference

### By Use Case
- **I'm new to Kubernetes** → Start with Phase 1
- **I know Kubernetes basics** → Start with Phase 2
- **I understand controllers** → Jump to Phase 4
- **I have an idea for a CRD** → Go directly to Phase 5's template

### By Time
- **5 minutes** → QUICK_REFERENCE.md
- **30 minutes** → Read 1 phase
- **2 hours** → Read all 5 phases
- **4 hours** → Read all 5 + do hands-on exercises

---

## 💬 Getting Help

### If You're Confused About:

| Topic | Where to Learn |
|-------|---|
| What is a CRD? | Phase 1 |
| How to write CRD YAML? | Phase 2 + Quick Reference |
| How controllers work? | Phase 3 + Phase 4 |
| What field to add to schema? | Phase 2 examples |
| RBAC for controllers? | Phase 4 + Quick Reference |
| Whether to use a CRD? | Phase 4 |
| How to design your CRD? | Phase 5 |

---

## 🏁 Your Journey Starts Here

You're about to understand one of Kubernetes' most powerful extension mechanisms.

### Next Step

→ **Open [PHASE_1_Fundamentals.md](PHASE_1_Fundamentals.md) and start reading!**

---

## 📚 Document Metadata

**Created**: March 30, 2026  
**Audience**: Kubernetes beginners who want to learn CRDs  
**Prerequisites**: Basic Kubernetes knowledge (Pods, Deployments, Services)  
**Estimated Time**: 3-5 hours for full learning plan  
**Status**: Comprehensive, ready for learning  

---

Good luck, and welcome to the world of Kubernetes extensibility! 🚀
