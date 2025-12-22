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
A Thor service instance that coordinates drones, scheduling, and job execution.

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
A control-plane component responsible for matching jobs to compatible nodes and dispatching commands to drones.

### Control Plane Data Store
A persistent store maintaining system state, including tenants, nodes, jobs, and related metadata.

---

## Execution Model

### Node Onboarding
Node onboarding registers a node with Thor and establishes its identity, ownership, and policy context.

During onboarding, the drone is provisioned with configuration and credentials required to connect to a hive. A node is registered but not schedulable until the drone connects and reports capabilities.

### Drone Connection and Registration
After onboarding, the drone establishes a persistent connection to a hive, authenticates, and registers the node.

Registration enables liveness tracking and confirms readiness to participate in scheduling and execution.

### Capability Discovery and Reporting
The drone inspects the node and reports its capabilities to the hive, which records them in the control plane data store.

A node becomes schedulable only after its capabilities are successfully reported and acknowledged.

### Idle State and Command Channel
Once schedulable, the drone enters an idle state and maintains an active command channel with the hive.

Through this channel, the hive issues commands such as asset acquisition, workload instantiation, or job execution. While idle, the drone continues reporting liveness and availability.

### Command Dispatch
The scheduler selects an idle, compatible node and dispatches a command to the corresponding drone.

Dispatch coordination ensures the node remains available at assignment time; failed dispatches result in rescheduling.

### Asset Acquisition
Before execution, the drone ensures all required assets are available on the node.

Asset access is constrained by tenant isolation and policy.

### Workload Instantiation
The drone ensures a suitable workload instance is available on the node, either by starting a new instance or reusing an existing one.

Instantiation prepares the runtime environment, applies configuration, and makes assets available to the workload.

### Job Execution
The drone submits the job to the workload instance and monitors execution until completion, failure, or termination.

### Status Reporting and Completion
During execution, the drone reports state transitions and progress to the hive.

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
From compatible candidates, the scheduler selects a node based on availability, load, quotas, and scheduling preferences.

Selection balances throughput and fairness while respecting priority.

### Dispatch and Reservation
The selected node receives a dispatch command via its drone.

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
The scheduler enforces isolation through tenant-scoped eligibility, quotas, and fairness controls.

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
Tenants authenticate when submitting jobs; drones authenticate to hives using onboarding credentials.

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
Job and node lifecycles are tracked through status updates recorded by the hive.

### Metrics and Monitoring
Control-plane components and drones emit metrics for scheduling activity, execution, and resource usage.

### Logging and Events
Key system events are recorded to provide a chronological record of activity and failures.

### Auditing
Security-relevant actions such as job submission, node onboarding, and policy decisions are auditable and retained per tenant.

## Non-Goals

The following items are explicitly out of scope for Thor at this stage:

### Geographic Affinity and Data Locality Guarantees

Thor does not currently provide guarantees or constraints related to geographic locality, regional affinity, or data residency. Hives may be deployed in multiple locations, but scheduling decisions are not inherently aware of geographic proximity unless explicitly modeled through capabilities or policy.

Any requirements related to regional placement, data sovereignty, or latency-sensitive execution must be expressed externally or through higher-level policy mechanisms.

### Network Topology Optimization

Thor does not optimize scheduling based on network topology, bandwidth, or latency between nodes, hives, or external systems.

### Workload Placement Optimization

Thor does not attempt to globally optimize workload placement for cost, performance, or energy efficiency beyond basic scheduling constraints and fairness policies.

### Workflow Semantics

Thor does not provide native support for multi-step workflows, dependency graphs, or orchestration of job pipelines. Such behavior may be implemented externally or layered on top of Thor.

