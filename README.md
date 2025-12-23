# Thor: Architectural Overview

## Scope
Thor is a distributed, multi-tenant orchestration platform for scheduling and executing AI workloads across heterogeneous nodes based on declared capabilities, policies, and availability.

---

## Core Concepts and Terminology

### Node
A physical or virtual computer equipped with AI-capable hardware and suitable for executing workloads.

### Drone
A software agent running on a node that connects to Thor, reports node capabilities, and executes assigned work.

### Hive
A Thor service instance that coordinates Drones, scheduling, and job execution.

### Tenant
A logical isolation boundary representing an organization or user group that owns workloads, policies, and resource quotas.

### Workload
A self-contained application that is able to perform a unit of computation.

### Asset
An external resource required by a workload at runtime, such as model weights or adapters, not embedded in the workload itself.

### Capability
A descriptive attribute of a node representing hardware, software, or operational characteristics relevant to workload execution.

Examples include:
- GPU model and architecture
- Number of GPUs
- Available GPU memory (VRAM)
- CPU cores and system memory
- Supported CUDA or driver versions
- Storage capacity or performance characteristics

### Job
A tenant-submitted request to execute a workload, represented as a payload with associated metadata.

### Job Metadata
Structured information associated with a job that guides scheduling, execution, and policy enforcement.

### Scheduler
A logical control-plane function responsible for matching jobs to compatible nodes and selecting execution targets.  
The scheduler is implemented as part of the Hive **Orchestrator**.

### Control Plane Data Store
A persistent store maintaining system state, including tenants, nodes, jobs, and related metadata.

---

## Execution Model

### Node Onboarding
Node onboarding registers a node with Thor and establishes its identity, ownership, and policy context.

During onboarding, the Drone is provisioned with configuration and credentials required to connect to a Hive. A node is registered but not schedulable until the Drone connects and reports capabilities.

### Drone Connection and Registration
After onboarding, the Drone establishes a persistent connection to a Hive, authenticates, and registers the node.

Registration enables liveness tracking and confirms readiness to participate in scheduling and execution.

### Capability Discovery and Reporting
The Drone inspects the node and reports its capabilities to the Hive, which records them in the control plane data store.

A node becomes schedulable only after its capabilities are successfully reported and acknowledged.

### Idle State and Command Channel
Once schedulable, the Drone enters an idle state and maintains an active command channel with the Hive.

Through this channel, the Hive issues commands such as asset acquisition, workload instantiation, or job execution. While idle, the Drone continues reporting liveness and availability.

### Command Dispatch
The Orchestrator selects an idle, compatible node and dispatches a command to the corresponding Drone.

Dispatch coordination ensures the node remains available at assignment time; failed dispatches result in rescheduling.

### Asset Acquisition
Before execution, the Drone ensures all required assets are available on the node.

Asset access is constrained by tenant isolation and policy.

### Workload Instantiation
The Drone ensures a suitable workload instance is available on the node, either by starting a new instance or reusing an existing one.

Instantiation prepares the runtime environment, applies configuration, and makes assets available to the workload.

### Job Execution
The Drone submits the job to the workload instance and monitors execution until completion, failure, or termination.

### Status Reporting and Completion
During execution, the Drone reports state transitions and progress to the Hive.

Upon completion, the final outcome is recorded in the control plane data store and node availability is updated.

---

## Scheduling Model

Thor schedules jobs by matching job metadata against node capabilities and current system state while enforcing tenant isolation, policies, and quotas.

### Eligibility and Scope
Scheduling begins by identifying nodes eligible for a job based on tenant association and policy.

Only registered, schedulable, and permitted nodes are considered.

### Capability Matching
Candidate nodes are filtered by evaluating whether reported capabilities satisfy required job constraints.

Only compatible nodes proceed to selection.

### Node Selection
From compatible candidates, the Orchestrator selects a node based on availability, load, quotas, and scheduling preferences.

Selection balances throughput and fairness while respecting priority.

### Dispatch and Reservation
The selected node receives a dispatch command via its Drone.

If dispatch fails, the job remains pending and may be rescheduled.

### Quotas and Fairness
Per-tenant quotas and fairness policies limit concurrency and resource consumption, particularly on shared nodes.

Tenant-associated nodes may be preferred for their owning tenant, subject to policy.

### Scheduling Outcomes
Scheduling results in one of the following:
- Job dispatched to a compatible node
- Job pending due to lack of available capacity
- Job rejected due to unsatisfiable constraints or policy

---

## Multi-Tenancy and Isolation Model

Thor supports multiple tenants sharing control-plane infrastructure while maintaining isolation across execution, scheduling, and state.

### Tenant Boundaries
All jobs, workloads, assets, and scheduling decisions are evaluated within a tenant context.

Tenants have no visibility into other tenants’ state unless explicitly permitted by policy.

### Node Ownership and Association
Nodes may be tenant-associated or shared. Ownership is established during onboarding and influences scheduling eligibility and preference.

### Workload and Asset Isolation
Workloads execute in isolated contexts. Asset access is tenant-scoped and limited to what is explicitly permitted by job metadata and policy.

### Scheduling Isolation
The Orchestrator enforces isolation through tenant-scoped eligibility, quotas, and fairness controls.

### Control Plane Isolation
Control-plane state is logically partitioned by tenant and protected by access controls.

---

## Failure Handling and Recovery

Thor is designed to tolerate failures while preserving system availability.

### Node and Drone Failures
Loss of connectivity is detected via missed heartbeats. Affected nodes are removed from scheduling, and executing jobs are marked failed or interrupted.

### Job Failure and Retry
Job failures are recorded and handled according to job metadata and policy, including retry, rescheduling, or permanent failure.

### Dispatch and Scheduling Failures
If dispatch fails or a node becomes unavailable before execution, scheduling is reevaluated.

### Control Plane Resilience
Persistent state enables control-plane components to recover from restarts without losing execution context.

---

## Security and Trust Model

Thor enforces security through identity, policy, and isolation across control and execution planes.

### Identity and Authentication
Tenants authenticate when submitting jobs; Drones authenticate to Hives using onboarding credentials.

### Authorization and Policy Enforcement
Authorization is based on tenant identity, job metadata, and system policy, and is enforced across scheduling, execution, and asset access.

### Node Trust and Classification
Nodes are classified during onboarding based on ownership and trust, influencing scheduling eligibility.

### Execution Isolation
Workloads run in isolated environments, limiting access strictly to permitted assets and configuration.

### Control Plane Protection
Control-plane data is protected through strict access controls and tenant-aware isolation.

---

## Observability and Auditing

Thor provides observability and auditing to support monitoring, debugging, and accountability.

### Execution Observability
Job and node lifecycles are tracked through status updates recorded by the Hive.

### Metrics and Monitoring
Control-plane components and Drones emit metrics for scheduling activity, execution, and resource usage.

### Logging and Events
Key system events are recorded to provide a chronological record of activity and failures.

### Auditing
Security-relevant actions such as job submission, node onboarding, and policy decisions are auditable and retained per tenant.

---

## Non-Goals

The following items are explicitly out of scope for Thor at this stage:

### Geographic Affinity and Data Locality Guarantees
Thor does not provide guarantees or constraints related to geographic locality, regional affinity, or data residency.

### Network Topology Optimization
Thor does not optimize scheduling based on network topology, bandwidth, or latency.

### Workload Placement Optimization
Thor does not globally optimize workload placement for cost, performance, or energy efficiency.

### Workflow Semantics
Thor does not provide native support for multi-step workflows or job dependency graphs.

---

## Bird’s-eye System Diagram
See `diagram.png` for a high-level system overview.

---

## Protocols

### drone <-> hive
A bidirectional gRPC stream used for command dispatch, status reporting, and liveness (non-exhaustive):

- **ping / pong**
- **get_capabilities / capabilities**
- **get_assets / assets / ensure_assets**
- **get_workloads / workloads / ensure_workloads**
- **execute_job / job_result **


### DevOps Endpoint
A REST API used by `thorctl` for operational control (non-exhaustive):

- `api/v1/tenants`
- `api/v1/jobs`
- `api/v1/nodes`
- `api/v1/tokens`

### Tenant-Facing Endpoint
- `POST api/v1/jobs` — submit a job and receive a result or error.

### Short-Running and Long-Running Jobs
The current design assumes short-running jobs. Long-running job support may introduce queue-backed execution without altering the core streaming model.

---

## Proposed Tech Stack

### Node
- Debian 13 (stable), x86_64
- systemd
- OpenSSH server
- containerd (or Docker Engine)
- NVIDIA CUDA support

### Control Plane Data Store
- PostgreSQL 17

### Control Plane Communication
- Bidirectional gRPC streaming between Hive and Drone

### Assets
- Object storage (e.g. S3)
- Cached on nodes

### Workload storage
- Container imager registry (e.g. ACR)

### Main Programming Language
- Go

#### Core Libraries
- gRPC / Protobuf (drone to hive communication)
- samber/do (lihtweight dependency injection toolkit)
- cobra (cli toolkit)
- logrus (structured logging package)
- gin (lightweight REST backend ramework)
- goose (database schema migrations)
- gorm (optional) (database ORM package, if needed)
- go-containerregistry (download/sync container images)
- containerd (spin containerd/docker containers)
- gopkg.in/yaml (yaml confuguration parsing)

---

## Produced Binary Artifacts

### drone
Node agent responsible for execution and reporting.

### hive
Single control-plane deployable providing orchestration, APIs, and coordination.

> May be split into multiple services in the future if required.

### thorctl
CLI tool for operators and automation.

> A web dashboard may be added later; CLI provides equivalent functionality initially.

## Configuration
Both hive and the drone configuration are in the human editable yaml files.

---
## Authentication
All actors in the system (tenants, devops staff, nodes) are authenticated. everyone is issued a JWT token which encapsulates actor's id and its role. Token are issued over the devops API endpoint, usually using the CLI tool. Federated authentication (Google, Azure) etc can be added later, if necessary.

---
## Transport-level security
All network communication (gRPC streams, REST endpoints, Postres connections) are protected by the TLS enabled transport. Certificate and key management are out of scope for this document

---

## Hive: Main Components

### Drone Manager
Manages Drone connections, authentication, and command channels.

### Orchestrator
Implements scheduling logic, node selection, dispatch coordination, and job state transitions.

### Job API
REST endpoint for job submission and inspection.

### Devops API
REST endpoint for devops operations 
- tenant management
- node management
- token management
