# Choosing Kubebuilder Over client-go for Our Operator

## Overview
We need to build an operator that:
- Watches multiple known CRDs (ArgoCD, ExternalSecrets, and internal CRDs).
- Logs status changes and publishes messages to Kafka.
- Has a small chance of monitoring *new* CRDs in the future.
- Should be maintainable, scalable, and secure.

This document compares **Kubebuilder** and **client-go** for our use case and explains why **Kubebuilder** is the better fit.

---

## Kubebuilder vs client-go

### **Kubebuilder** (built on controller-runtime)

| Pros | Cons |
|------|------|
| **Standard design**: Widely used, well-structured code layout makes onboarding new developers easy. | Slightly opinionated framework — adds abstractions that may feel heavy for purely generic watchers. |
| **Easy factory pattern**: Supports running multiple managers and reconcilers in a single binary. | Dynamic watching of completely unknown CRDs requires extra coding (but possible). |
| **Race condition safety**: Built-in helpers for `.Status` updates avoid common race conditions. |  |
| **Scalable**: Can evolve from simple status logging to complex reconciliation logic without architectural changes. |  |
| **TLS modularity**: Supports secure webhooks and API servers if needed in the future. |  |
| **RBAC generation**: Generates least-privilege RBAC manifests from code annotations. |  |
| **Testing support**: Has `envtest` for local integration testing without a cluster. |  |
| **Built-in lifecycle management**: Leader election, health checks, graceful shutdown, caching. |  |

---

### **client-go** (low-level Kubernetes API client)
| Pros | Cons |
| :--- | :--- |
| **Full flexibility**—you control informers, queues, and event handlers directly. | **No scaffolding**—must hand-roll managers, leader election, RBAC, caching, and testing setups. |
| **Best suited for lightweight, highly dynamic CRD watching** without pre-defined types. | **More boilerplate**—repetitive informer wiring and lifecycle code. |
| | **Harder for new developers** to read and maintain. |
| | **No built-in safety nets** for status update race conditions. |

---

## Why We Prefer Kubebuilder for Our Case

1. **Most CRDs are known today** → Kubebuilder’s typed controllers give compile-time safety.
2. **Future flexibility** → We can still add a dynamic watcher using `client-go` inside a Kubebuilder project.
3. **Maintainability** → Standardized structure improves readability for new hires.
4. **Factory pattern support** → Multiple managers and reconcilers can be easily created.
5. **Built-in safeguards** → Avoids status update race conditions with built-in helpers.
6. **Growth path** → If we need complex reconciliation later, the architecture already supports it.
7. **Security features** → TLS modularity and RBAC generation are already integrated.

---

## Recommendation
Use **Kubebuilder** as the base framework, with:
- **Typed controllers** for known CRDs.
- **A generic dynamic watcher** for rare new CRDs.
- This hybrid approach balances maintainability, safety, and future flexibility.
