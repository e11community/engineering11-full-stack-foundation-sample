```
███████╗ ██╗ ██╗
██╔════╝███║███║
█████╗  ╚██║╚██║
██╔══╝   ██║ ██║
███████╗ ██║ ██║
╚══════╝ ╚═╝ ╚═╝
```

[engineering11.com](https://engineering11.com)

# Backend — Engineering11 Full Stack Foundation

This is a **sample/mock backend repository** demonstrating the structure and capabilities of the Engineering11 Full Stack Foundation.

It represents a typical backend codebase composed of both:

- **Engineering11-provided platform microservices**, each included with a `README.md` describing its purpose and capabilities, and
- **Customer-built microservices**, implemented using the same Engineering11 design patterns and service structure.

The repository includes a working example of a custom microservice alongside documentation folders for each available Engineering11 platform service, providing a concrete reference for how platform services and product-specific services coexist within a single system.

---

## Repository Structure

```
├── microservices/                 # All backend microservices (platform + custom)
│   ├── service-user/              # Engineering11 platform microservice
│   ├── service-auth/              # Engineering11 platform microservice
│   ├── service-messaging/         # Engineering11 platform microservice
│   ├── service-notifications/     # Engineering11 platform microservice
│   ├── service-files/             # Engineering11 platform microservice
│   ├── service-customer/          # Engineering11 platform microservice
│   ├── service-example/           # Sample custom microservice built using E11 patterns
│   └── ...                        # Additional platform and customer services
```

All backend microservices — whether provided by Engineering11 or built by the customer — live in the same directory and follow the same structural and operational conventions. This reflects how real Engineering11-backed systems are organized and evolved over time.

---

## Typical Backend Structure for an Engineering11-Based System

This repository represents a complete Engineering11 backend codebase. Platform microservices and customer-built services live side by side, supported by shared libraries, operational tooling, and standardized workflows.

```
backend-repo/
├─ microservices/                 # All E11 provided and custom backend services
│                                 # Each service follows shared E11 patterns

├─ packages/
│   └─ shared/                    # Shared server-side libraries used across services
├─ static/                        # Static assets package for defining permissions, products, notifications and more
├─ rules/                         # Data access and security rules with tests
├─ ops/                           # Operational tooling (bin, env helpers, platform scripts, playbooks)
├─ queues/                        # Async task / background job queue definitions
├─ env/
│   └─ README.md                  # Environment setup and configuration notes
├─ cloudbuild.yaml                # CI build entrypoint
├─ deploy-*.sh                    # Deployment scripts
├─ docker-compose*.yaml           # Local development orchestration
├─ firebase.json                  # Platform configuration (If Firestore is used)
├─ firestore.*                    # Data store configuration and rules (If Firestore is used)
├─ nx.json                        # Task runner configuration with caching
├─ package.json                   # Yarn workspace root
├─ yarn.lock                      # Workspace lockfile (services, packages, rules, static)
└─ release-please-config.json     # Multi-package versioning and release automation
```

This structure reflects how real Engineering11-backed systems are built and operated: a single, cohesive backend workspace where platform services and product-specific services evolve together using the same tooling, patterns, and operational conventions.

This repository uses **Yarn workspaces**, **Nx**, and **release-please** to improve the day-to-day engineering experience.

Yarn workspaces provide a single, consistent dependency graph across services and shared packages.  
Nx enables fast, cache-aware task execution and clear project boundaries across a large multi-service codebase.  
Release-please automates versioning and releases across multiple services and packages in a predictable, low-friction way.

Together, these tools reduce build times, simplify dependency management, and make it easier for teams to work efficiently as the system scales.

---

## Backend Architecture

Engineering11 backends are cloud-native, event-driven, and designed for scale:

- **Containerized microservices** — independently deployable REST APIs and workers
- **Event-driven workflows** — async processing via pub/sub messaging
- **Repository-pattern data access** — consistent interfaces across storage providers
- **Multi-tenant by default** — data partitioning and isolation built into every layer
- **Centralized observability** — structured logging, monitoring, and tracing

Services follow consistent patterns to reduce cognitive load and operational complexity.

---

## Microservice Anatomy

Engineering11 microservices follow a consistent, finite set of structured parts. Each service includes only the components it needs for its domain, but those components always follow the same patterns and conventions:

- **Server APIs** — published service packages that other backend services can install to access shared models, domain logic, and repository-based data access through stable, versioned interfaces.
- **IPC REST APIs** — authenticated, service-to-service REST endpoints used for internal communication and inter-process coordination between backend services.
- **REST APIs** — externally exposed, client-facing endpoints consumed by frontend applications and integrated third-party systems.
- **Background jobs** — asynchronous and long-running processes executed outside of request/response flows.
- **Task handlers** — discrete asynchronous handlers responsible for processing queued tasks and workflow steps.
- **Shared full-stack libraries** — reusable models, utilities, and primitives shared across backend services and frontend applications to ensure consistency.
- **Migrations** — controlled change sets used to evolve databases and related system state in a predictable and repeatable way.

This structure makes services predictable to work in, review, and operate over time.

---

## What You Build On Top

Engineering11 provides the foundational design patterns and building blocks — your team uses them to build the features that differentiate your product.

With the platform in place, your team owns and implements:

- Product-specific business logic and domain workflows
- Custom API endpoints, contracts, and data models
- Event consumers and producers tailored to your use cases
- AI services, enrichment pipelines, and decision logic
- Integration logic with external systems and partners

Engineering11 accelerates development by standardizing the foundation, without hiding the architecture — your engineers retain full visibility, control, and ownership of the system.

---

## Operational Philosophy

The backend should be **boring to operate**.

Predictable deployments, strong observability, and safe scaling mean your team spends time building features — not firefighting infrastructure.
