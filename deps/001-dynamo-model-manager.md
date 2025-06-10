# Dynamo Model Manager

**Status**: Under Review

**Authors**: Kavin Krishnan, Nicolas Noble, Zhongdongming Dai

**Category**: Architecture

**Required Reviewers**: Itay Neeman, Ganesh Kudleppanavar, Neelay Shah,
Maksim Khadkevich, Julien Mancuso, Ishan Dhanani, Kyle Kranen

**Review Date**: 06/11/2025

**Implementation PR / Tracking Issue**: WIP

## Problem Statement

Dynamo workers require access to model files stored in repositories. The architecture supports multiple backend frameworks, with file type compatibility determined by the chosen backend. Currently, model files can be downloaded from various cloud storage services or accessed directly from the local filesystem.

With model files becoming larger and cluster sizes growing, having every node fetch files from the cloud results in:
  - Increasing ingress flow
  - Higher latency for backend startup
  - Inefficient resource utilization

We propose a distributed proxy system that:
  - Downloads files from the same repositories as the base Triton Server
  - Caches them locally
  - Allows local cluster nodes to download from cache instead of external repositories

## Goals

### Primary Goals
- **Startup Time Reduction**: Drastically decrease median startup time for Dynamo workers fetching model files from external servers
- **Ingress Bandwidth Reduction**: Significantly reduce total ingress bandwidth by using a single inner data fetching node instead of multiple nodes
- **Versioning**: Automatically detect and update local cache when new model file versions are available
- **TensorRT Models Pre-compilation**: Support pre-compilation of downloaded models for various TensorRT formats to further speed up start-up time

### Future Goals
- **Storage**: Potential for custom storage system leveraging locality and hardware
- **Encryption**: Support for asset encryption for private model files
- **Optimized Model Sharing**: Utilize NIXL library to reduce latency for sharing models between nodes
- **Fast Checkpoint Transfer**: Optimize transfer between RL workloads and inference deployment with the NIXL library

### Non-Goals
- **Optimized Routing**: Load balancing, network throttling, and advanced network features
- **Security**: No additional security beyond existing file validation and authentication mechanisms

## Requirements

### Distributed Storage
- Uses Container Storage Interface (CSI) for storage system flexibility
- Requires RWX support for cluster-wide read/write access
- Uses content hashing for idempotent file operations

### Networking
- Utilizes UCX framework for client-to-pod communication
- Optimized for HPC networking

### Load Balancing
- Proxy server is nearly stateless
- Implements pub-sub mechanism for concurrent downloads
- Handles thundering herd effect during cluster startup

### Caching Tiers
1. Distributed storage for cold caching
2. Per-pod in-memory caching (faster, shorter TTL)
3. Local client cache in Dynamo nodes

### Database Requirements
Minimal information storage:
- Original asset file URL
- Download timestamp
- Last-Modified header
- Content hash value
- Pub-sub channel ID (if downloading)
- Pod ID with in-memory cache

### Third-Party Compatibility
Initially, enhancement should support [Fluid](https://fluid-cloudnative.github.io/) as a data storage for models.

There is already an example of [Fluid integration with Dynamo](https://github.com/ai-dynamo/dynamo/blob/main/docs/guides/dynamo_deploy/model_caching_with_fluid.md) in the main repository.

We will be open to other data orchestrators down the roadmap.

## Architecture

### Components
1. **Service Cluster**
   - Handles caching and downloading of models
   - Implements distributed proxy functionality
   - Manages cluster-wide operations

2. **Client Library**
   - Embedded within servers
   - Multi-language support
   - Uses UCX for data plane
   - Uses gRPC for control plane

### Distribution
- Distributed as Kubernetes manifest
- Includes deployments and operators for stateless and stateful components

## Alternative Solutions

### Partner Approaches
1. **JuiceFS Distributed Cache**
   - Used by BentoML for LLM loading
   - Generic solution lacking specialized model features

2. **KServe ModelMesh**
   - Inference-centric model serving service
   - Lacks hardware-level acceleration and TensorRT support
   - Java-based

3. **Amazon SageMaker Multi-Model Caching**
   - Similar functionality
   - Limited to Amazon ecosystem

4. **Hugging Face Datasets Cache**
   - Partial solution
   - Not designed for large-scale distribution

### Related Technologies
- [Run AI Model Streamer](https://www.run.ai/blog/accelerating-model-loading-with-run-ai-model-streamer)
- [NVIDIA NIM](https://developer.nvidia.com/nim)
- [NVIDIA NIM Operator Cache](https://docs.nvidia.com/nim-operator/latest/cache.html)
- [NVIDIA NeMo Datastore](https://docs.nvidia.com/nim-operator/latest/data-store.html)