<!--
# SPDX-FileCopyrightText: Copyright (c) 2025 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
-->
# AIPerf Kubernetes Deployment Enhancement

**Status**: Draft

**Authors**: Anthony Casagrande

**Category**: Architecture

**Replaces**: None

**Replaced By**: None

**Sponsor**: Ganesh Kudleppanavar

**Required Reviewers**: Ganesh Kudleppanavar, Nicolas Noble, Biswa Ranjan Panda, Neelay Shah, Hannah Zhang, Itay Neeman, Maksim Khadkevich

**Review Date**: [Date for review]

**Pull Request**: [Link to Pull Request of the Proposal itself]

**Implementation PR / Tracking Issue**: [Link to Pull Request or Tracking Issue for Implementation]

# Summary

This proposal outlines the enhancement of AIPerf to support distributed deployment on Kubernetes clusters. The enhancement enables AIPerf to generate significantly higher concurrent loads by distributing work across multiple pods in a Kubernetes cluster, overcoming single-node performance limitations. The solution adopts a true per-service pod architecture where each AIPerf service runs in its own dedicated pod, enabling independent scaling and resource allocation.

AIPerf currently supports only single-node multiprocess deployment. This enhancement proposes implementing the existing `KubernetesServiceManager` stub to enable distributed deployment while maintaining full compatibility with existing service management patterns, ZMQ communication protocols, and configuration systems.

# Motivation

The current AIPerf implementation is limited to single-node execution, which constrains the maximum concurrent load that can be generated for benchmarking AI inference services. A critical limitation of single-node deployments is the exhaustion of ephemeral ports, which limits concurrent connections to approximately 65k per node (HTTP2 can increase this, but at some point other limits become relevant). As AI inference workloads scale to serve production traffic, there is a critical need to validate performance under extremely high concurrency conditions. Single-node AIPerf deployments cannot achieve these concurrency levels due to operating system networking constraints, hardware limitations, and the inherent scalability limits of running all services on a single machine.

Dynamo inference services are deployed on Kubernetes clusters and need to be validated against realistic, high-concurrency load patterns that exceed single-node capabilities. The lack of distributed load generation capability prevents teams from conducting comprehensive performance validation before deploying to production, potentially leading to service degradation or failures under actual concurrent user load.

## Goals

* Enable distributed load generation across multiple Kubernetes pods to achieve high concurrency levels (1M+ connections)
* Overcome single-node ephemeral port limitations (~65K) by distributing connections across multiple worker pods
* Maintain compatibility with existing AIPerf CLI interfaces and configuration options
* Provide simple deployment experience through single CLI flag (`--kubernetes`)
* Leverage existing ZMQ TCP support for distributed communication (minimal code changes)
* Enable automatic scaling of worker pods based on target concurrency

### Non Goals (MVP)

* Automatic pod failure recovery and restart
* Supporting non-Kubernetes container orchestration platforms
* Providing a web-based UI for job management and visualization
* Implementing cross-cluster or multi-cloud distributed deployments
* Replacing the existing single-node deployment mode (both modes coexist)
* Advanced security features (mTLS, network policies, secret management)

## Requirements

### REQ 1 Distributed Architecture Support

AIPerf **MUST** support deployment of its services across multiple Kubernetes pods using a true per-service pod architecture while maintaining the existing service communication patterns. Each AIPerf service (system controller, dataset manager, timing manager, records manager, worker manager, worker, record processor) **MUST** run in its own dedicated pod. Worker and record processor services **MUST** be deployable as independently scalable pod replicas.

### REQ 2 Kubernetes API Integration and Compatibility

The implementation **MUST** use the Kubernetes API directly for pod orchestration rather than relying on external tools. The `KubernetesServiceManager` **MUST** implement the existing interfaces to ensure compatibility with AIPerf's service management architecture.

### REQ 3 Concurrency Scaling

The distributed deployment **MUST** be capable of sustaining at least 1M concurrent connections when deployed with sufficient pod replicas. The system **MUST** support dynamic scaling of worker pods based on target concurrency requirements and overcome single-node ephemeral port limitations.

### REQ 4 Communication Protocol Compatibility

The distributed implementation **MUST** maintain compatibility with existing ZMQ-based communication patterns. All inter-service communication **MUST** function correctly across pod boundaries using the same message protocols.

### REQ 5 Deployment Lifecycle Management

The system **MUST** support complete lifecycle management including deployment, configuration, execution, and cleanup of Kubernetes resources. Failed deployments **SHOULD** be automatically cleaned up to prevent resource leaks.

### REQ 6 Developer Experience and Simplicity

The implementation **MUST** provide a simple, intuitive deployment experience that allows developers to achieve 1M+ concurrent connections with minimal configuration. The system **SHOULD** follow basic Kubernetes security practices while prioritizing ease of use and performance over enterprise complexity.


# Proposal

## Architecture Overview

The proposed Kubernetes deployment architecture adopts a true per-service pod approach where each AIPerf service runs in its own dedicated pod. This design provides maximum flexibility for independent scaling, resource allocation, and fault isolation. The architecture separates AIPerf services into two categories:

1. **Singleton Service Pods**: Core coordination services that run as single instances
2. **Scalable Service Pods**: Services that scale horizontally based on load requirements

### Service Pod Architecture

#### System Controller (Single Pod)
- Orchestrates the entire benchmark lifecycle and coordinates other services
- Manages the ProxyManager with embedded ZMQ proxies for message routing

#### Timing Manager (Single Pod)
- Issues credits (request tokens) and controls benchmark timing

#### Records Manager (Single Pod)
- Collects and aggregates metrics from distributed record processors

#### Worker Manager (Single Pod)
- Coordinates worker scaling and management operations

#### Dataset Manager (Single Pod)
- Manages dataset loading and distribution across the cluster

#### Worker (Scalable Pods)
- Execute inference requests against target endpoints
- Scale horizontally based on target concurrency requirements (1 to N replicas)
- Each pod can handle concurrent connections up to the configured `AIPERF_HTTP_CONNECTION_LIMIT`

#### Record Processor (Scalable Pods)
- Perform request and response tokenization and initial metrics processing
- Scale horizontally based on response processing throughput and concurrent request volume (1 to N replicas)
- Send processed metrics back to the records manager via ZMQ

## Communication Architecture

The existing ZMQ-based communication patterns are preserved with the following adaptations for distributed deployment:

### ZMQ Communication Flow
The existing ZMQ communication uses multiple proxy patterns managed by the ProxyManager:

1. **Credit Distribution**: Timing manager pushes credits through CREDIT_DROP address, workers pull via PUSH/PULL
2. **Credit Return**: Workers return completed credits through CREDIT_RETURN address via PUSH/PULL
3. **Raw Inference Results**: Workers send raw responses through RAW_INFERENCE_PROXY to record processors via PUSH/PULL proxy
4. **Processed Records**: Record processors send processed metrics to records manager via RECORDS address using PUSH/PULL
5. **Event Bus**: System controller and services communicate via EVENT_BUS_PROXY using XPUB/XSUB proxy for coordination messages
6. **Dataset Requests**: Workers request conversation data from dataset manager via DATASET_MANAGER_PROXY using DEALER/ROUTER proxy

### Network Configuration
- System controller pod exposes ZMQ proxy ports via Kubernetes services
- All service pods connect to system controller services using Kubernetes DNS
- Each singleton service pod exposes its own service endpoint for direct communication
- All ZMQ communication uses TCP transport

## Deployment Modes

### CLI-Driven Deployment (MVP)

**Quick Start Example:**

```bash
# Run 100K concurrent connections against inference service
aiperf profile \
  --kubernetes \
  --endpoint-type chat \
  --streaming \
  -u http://my-llm-service.default.svc.cluster.local:8080 \
  -m my-llm-model \
  --concurrency 100000 \
  --duration 300 \
  --public-dataset sharegpt

# AIPerf automatically:
# 1. Creates namespace: aiperf-<timestamp>
# 2. Deploys System Controller pod to cluster
# 3. System Controller deploys:
#    - 4 singleton service pods (dataset, timing, records, worker manager)
#    - 200 worker pods (100K / 500 connections per worker)
#    - 50 record processor pods
# 4. Runs benchmark for 5 minutes
# 5. Retrieves results to local ./artifacts/ directory
# 6. Cleans up all Kubernetes resources
# 7. Displays metrics summary in terminal
```

**Advanced Options:**

```bash
# Use custom namespace (won't auto-delete)
aiperf profile --kubernetes --kubernetes-namespace my-benchmark ...

# Use custom kubeconfig (defaults to ~/.kube/config)
aiperf profile --kubernetes --kubeconfig ~/.kube/prod-cluster ...
```

## Resource Management

### Pod Resource Allocation

To be determined based on performance testing. Example resource allocation:

**Singleton Services** (1 replica each):
- **System Controller**: 2 CPU, 2Gi memory (hosts ZMQ proxies, minimal overhead)
- **Dataset Manager**: 2 CPU, 2Gi memory (distributes conversations via ZMQ)
- **Timing Manager**: 1 CPU, 1Gi memory (credit distribution)
- **Records Manager**: 1 CPU, 2Gi memory (metrics aggregation)
- **Worker Manager**: 1 CPU, 1Gi memory (health monitoring)

**Scalable Services**:
- **Worker Pods**: 2 CPU, 2Gi memory per pod
  - Recommended: 500 concurrent connections per worker
  - Maximum: 2,500 connections per worker (set via `AIPERF_HTTP_CONNECTION_LIMIT`)
- **Record Processor Pods**: 2 CPU, 2Gi memory per pod
  - Ratio: 1 record processor per 4 worker pods

# Implementation Details

## Kubernetes Service Manager

The `KubernetesServiceManager` implements the same `ServiceManagerProtocol` interface as `MultiProcessServiceManager`, but deploys services as Kubernetes pods instead of processes.

**Core Responsibilities:**
- Deploy service pods using Kubernetes API
- Manage Kubernetes services for ZMQ communication endpoints
- Track pod lifecycle and handle cleanup
- Same interface as multiprocess mode

## ZMQ Network Configuration

AIPerf already has ZMQ TCP support. For Kubernetes, the ZMQ host is configured to use the System Controller's Kubernetes service DNS name instead of localhost.

### Service Exposure
The system controller pod exposes ZMQ proxy endpoints via Kubernetes ClusterIP service:

```yaml
# System Controller Service (ZMQ Proxies)
apiVersion: v1
kind: Service
metadata:
  name: aiperf-system-controller
  namespace: aiperf
spec:
  selector:
    app: aiperf-system-controller
  type: ClusterIP
  ports:
    # Using existing port numbers from ZMQTCPConfig
    - name: credit-drop
      port: 5562
      targetPort: 5562
    - name: credit-return
      port: 5563
      targetPort: 5563
    - name: records
      port: 5557
      targetPort: 5557
    # Dataset Manager Proxy
    - name: dataset-manager-proxy-frontend
      port: 5661
      targetPort: 5661
    - name: dataset-manager-proxy-backend
      port: 5662
      targetPort: 5662
    # Event Bus Proxy
    - name: event-bus-proxy-frontend
      port: 5663
      targetPort: 5663
    - name: event-bus-proxy-backend
      port: 5664
      targetPort: 5664
    # Raw Inference Proxy
    - name: raw-inference-proxy-frontend
      port: 5665
      targetPort: 5665
    - name: raw-inference-proxy-backend
      port: 5666
      targetPort: 5666
```

### Connection Configuration
Service pods use Kubernetes DNS to connect to the System Controller service (e.g., `aiperf-system-controller.aiperf.svc.cluster.local`) for all ZMQ communication.

## CLI Orchestration and RBAC

The AIPerf CLI uses the Kubernetes Python API for all cluster operations:
1. Creating namespace and RBAC resources via API
2. Deploying System Controller pod with benchmark configuration via API
3. Monitoring benchmark progress and streaming logs via API and ZMQ TCP
4. Retrieving results from Records Manager pod via API
5. Cleaning up all resources on completion via API

### RBAC Resource Creation
```python
# ServiceAccount for AIPerf operations
service_account = {
    "apiVersion": "v1",
    "kind": "ServiceAccount",
    "metadata": {
        "name": "aiperf-service-account",
        "namespace": namespace
    }
}

# ClusterRole with required permissions (following least-privilege principle)
cluster_role = {
    "apiVersion": "rbac.authorization.k8s.io/v1",
    "kind": "ClusterRole",
    "metadata": {"name": "aiperf-cluster-role"},
    "rules": [
        {
            "apiGroups": [""],
            "resources": ["pods", "services", "configmaps"],
            "verbs": ["create", "get", "list", "watch", "update", "patch", "delete"]
        },
        {
            "apiGroups": ["apps"],
            "resources": ["deployments", "replicasets"],
            "verbs": ["create", "get", "list", "watch", "update", "patch", "delete"]
        },
    ]
}

# ClusterRoleBinding to associate ServiceAccount with ClusterRole
cluster_role_binding = {
    "apiVersion": "rbac.authorization.k8s.io/v1",
    "kind": "ClusterRoleBinding",
    "metadata": {"name": "aiperf-cluster-role-binding"},
    "subjects": [{"kind": "ServiceAccount", "name": "aiperf-service-account", "namespace": namespace}],
    "roleRef": {"kind": "ClusterRole", "name": "aiperf-cluster-role", "apiGroup": "rbac.authorization.k8s.io"}
}
```

#### Implementation Approach
The CLI uses the Kubernetes Python client to create all resources programmatically via the API. The client automatically uses the kubeconfig file (`~/.kube/config` by default) for authentication and cluster access.

## Dataset Distribution Strategy

### Dataset Upload to Cluster

The CLI transfers dataset files to the cluster using one of these approaches:

1. **File-based Datasets**: Streamed to DatasetManager or System Controller pod via Kubernetes API (file upload)
2. **Public Datasets**: DatasetManager downloads directly from URL (no upload needed)
3. **Synthetic Datasets**: DatasetManager generates synthetic conversations

The implementation will determine the optimal transfer method based on dataset size and type.

### Dataset Distribution to Workers

**Current Challenge:**
- DatasetManager distributes conversations to worker pods via ZMQ DEALER/ROUTER
- High request volume could create bottleneck at DatasetManager
- ZMQ request/reply latency scales with worker count

**Optimization Strategies (To Be Evaluated):**

1. **Worker-Side Caching/Pre-Fetching**: Workers pre-fetch and cache conversations, request new ones less frequently
2. **Batch Distribution**: DatasetManager sends batches of conversations per request instead of one-at-a-time
3. **Multiple DatasetManager Replicas**: Scale DatasetManager horizontally with dataset sharding
4. **Redis Cache**: DatasetManager caches conversations in Redis for workers to fetch
5. **Shared Volumes**: DatasetManager mounts shared volumes to worker pods for dataset distribution

**MVP Approach:**
- Single DatasetManager pod with in-memory dataset
- Workers request conversations via ZMQ DEALER/ROUTER as needed
- Performance testing will determine if optimization is needed for high worker counts

## Configuration Management

### CLI Parameters
**Kubernetes Deployment Parameters**:
- `--kubernetes`: Enable Kubernetes deployment mode
- `--kubernetes-namespace`: Target Kubernetes namespace (optional, defaults to auto-generated namespace)
- `--kubeconfig`: Path to Kubernetes configuration file (optional, defaults to `~/.kube/config`)

**Namespace Behavior**:
- If `--kubernetes-namespace` is specified, AIPerf will use that namespace
- If not specified, AIPerf will auto-generate a unique namespace (e.g., `aiperf-<timestamp>` or `aiperf-<job-id>`)
- Auto-generated namespaces are automatically cleaned up after benchmark completion

**Existing Parameters (Work with Both Deployment Modes)**:
- `--record-processors`: Controls record processor scaling
- `--workers-max` / `--max-workers`: Controls worker scaling

## Container Images

A single container image supports all AIPerf service modes. Service type is determined by environment variables passed to the pod, using the existing code to instantiate the appropriate service class.

## Artifact and Export File Retrieval

AIPerf generates output files including metrics exports (JSON, CSV) and logs that users need to access after benchmark completion. In the Kubernetes deployment, these files are generated by the Records Manager pod and must be retrieved to the user's local filesystem via the Kubernetes Python API.

### Basic Approach

The Records Manager pod acts as the central collection point for all benchmark artifacts. It receives processed metrics from distributed pods and generates the final export files in its local artifact directory using the existing AIPerf exporters.

After benchmark completion, AIPerf automatically retrieves all artifact files from the Records Manager pod to the user's local filesystem using the Kubernetes Python API. This preserves the standard AIPerf directory structure and provides users with the same export files they would receive from a single-node deployment.

Once files are successfully retrieved, all Kubernetes pods are automatically cleaned up to minimize cluster resource usage.

# Implementation Plan

## MVP Scope

**Success Criteria:**
- Sustain 100K concurrent connections for 5+ minutes
- Results match single-node quality (±5% variance)
- Complete artifact retrieval in <60 seconds
- Successful cleanup of all resources
- Zero manual intervention required for happy path

**Main Components:**
1. Implement `KubernetesServiceManager` (create pods via Kubernetes API instead of processes)
2. CLI Kubernetes mode (namespace creation, pod deployment, monitoring via API)
3. Configuration serialization (pass config to pods via ConfigMap/env vars)
4. Artifact retrieval (stream results from pods to local via Kubernetes API)

# Architecture Diagram (MVP)

```
┌──────────────────────────────────────────────────────────────────┐
│ User's Local Machine                                             │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ AIPerf CLI (Python)                                        │  │
│  │  - Reads ~/.kube/config                                    │  │
│  │  - Creates namespace, RBAC, ConfigMap via K8s API          │  │
│  │  - Deploys System Controller pod via K8s API               │  │
│  │  - Monitors via Kubernetes API and ZMQ TCP                 │  │
│  │  - Retrieves results via Kubernetes API                    │  │
│  └────────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              │ Kubernetes Python API             │
└──────────────────────────────┼───────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│ Kubernetes Cluster (aiperf-<timestamp> namespace)                │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ System Controller Pod                                      │  │
│  │  - Runs ProxyManager (ZMQ proxies)                         │  │
│  │  - Runs KubernetesServiceManager                           │  │
│  │  - Exposes ZMQ ports via Service (5557, 5562, 5563...)     │  │
│  └────────────────────────────────────────────────────────────┘  │
│                              │                                   │
│              ┌───────────────┼───────────────┐                   │
│              │               │               │                   │
│              ▼               ▼               ▼                   │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐              │
│  │ Dataset Mgr  │ │ Timing Mgr   │ │ Records Mgr  │              │
│  │ Pod          │ │ Pod          │ │ Pod          │              │
│  │ - DEALER/    │ │ - Issues     │ │ - Aggregates │              │
│  │   ROUTER     │ │   credits    │ │   metrics    │              │
│  │   via ZMQ    │ │ - PUSH/PULL  │ │ - PUSH/PULL  │              │
│  └──────────────┘ └──────────────┘ └──────────────┘              │
│                              │                                   │
│              ┌───────────────┼───────────────┐                   │
│              ▼               ▼               ▼                   │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │ Worker Pods (200 replicas for 100K concurrency)          │    │
│  │  - Pull credits from TimingManager                       │    │
│  │  - Request conversations from DatasetManager             │    │
│  │  - Make HTTP requests to target inference service        │    │
│  │  - Push raw results to RecordProcessors                  │    │
│  │  - Each handles 500 concurrent connections               │    │
│  └──────────────────────────────────────────────────────────┘    │
│                              │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │ Target Inference Service (user's LLM endpoint)           │    │
│  │  - Receives HTTP requests from 200 worker pods           │    │
│  │  - Returns streaming or non-streaming responses          │    │
│  └──────────────────────────────────────────────────────────┘    │
│                              │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │ Record Processor Pods (50 replicas)                      │    │
│  │  - Pull raw results from workers                         │    │
│  │  - Tokenize requests/responses                           │    │
│  │  - Calculate metrics                                     │    │
│  │  - Push to RecordsManager                                │    │
│  └──────────────────────────────────────────────────────────┘    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

# Alternate Solutions

## Hybrid Co-location Approach

**Pros:**

* Reduces pod count by co-locating coordination services
* Simpler service discovery with fewer network endpoints
* Potentially lower networking overhead between co-located services
* Easier resource management for singleton services

**Cons:**

* Mixed approach complicates deployment logic (some services co-located, others separate)
* Less flexibility for independent resource tuning of singleton services
* Reduced fault tolerance as controller pod failure affects multiple services
* Harder to debug individual service issues when co-located

**Reason Rejected:**

* Per-service pods provide maximum flexibility and isolation
* Resource requirements for singleton services are sufficiently small that co-location benefits are minimal
* Consistency of having all services in individual pods simplifies architecture

## Kubernetes Job-Based Deployment Only

**Pros:**

* Leverages native Kubernetes job scheduling and lifecycle management
* Integrates well with CI/CD pipelines and batch processing workflows
* Automatic cleanup and resource management through Kubernetes job controller
* Simple YAML-based configuration without custom CLI extensions

**Cons:**

* Limited real-time control and monitoring during benchmark execution
* Difficult to implement dynamic scaling based on runtime metrics
* Reduced flexibility for interactive benchmark sessions
* May not support complex coordination patterns required by AIPerf

**Reason Rejected:**

* Meeting discussion indicated desire for both interactive CLI-driven deployment and job-based execution
* Constraints of Kubernetes job model may limit AIPerf's coordination capabilities
* Decision made to support CLI-driven deployment as primary interface


## External Orchestrator with Helm Charts

**Pros:**

* Leverages mature Helm ecosystem for templating and package management
* Easier deployment across different Kubernetes distributions
* Built-in versioning and rollback capabilities through Helm
* Community-standard approach for complex Kubernetes applications

**Cons:**

* Adds external dependency on Helm installation and management
* More complex CLI implementation to generate and apply Helm charts
* Additional learning curve for users unfamiliar with Helm
* May complicate custom resource management and cleanup logic

**Reason Rejected:**

* Meeting emphasized using Kubernetes API directly rather than external tools
* Adds complexity without addressing core scaling requirements
* Direct API approach provides more control over resource lifecycle management

# Background

AIPerf is NVIDIA's AI inference performance benchmarking tool designed to validate and stress-test AI inference services. The current implementation uses a multi-service architecture where services communicate via ZeroMQ (ZMQ) message passing. Services include the system controller (orchestration), dataset manager (data loading), timing manager (request rate control), workers (request execution), and record processors (response processing and metrics).

## References

* [Kubernetes API Documentation](https://kubernetes.io/docs/reference/kubernetes-api/)
* [ZeroMQ Distributed Messaging Guide](https://zguide.zeromq.org/docs/chapter4/)
* [Kubernetes Deployment and ReplicaSet Concepts](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

## Terminology & Definitions

| Term | Definition |
| :---- | :---- |
| **Credit** | Token representing a single-turn or multi-turn conversation to be executed by a worker |
| **Ephemeral Port** | Temporary port assigned by the OS for outbound connections, limited to ~65k per node |
| **ZMQ Proxy** | Service that routes messages between distributed ZeroMQ clients using various patterns |

## Acronyms & Abbreviations

**CRD:** Custom Resource Definition
**RBAC:** Role-Based Access Control
**ZMQ:** ZeroMQ Messaging Protocol
