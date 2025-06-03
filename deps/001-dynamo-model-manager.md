# Model Manager for Triton Server

## Problem Statement

When the Triton Server runs, it needs to access model files organized in repositories. The architecture is backend-agnostic, with file types restricted only by the framework backend used. The current mechanism allows downloading model files from various cloud services or using the local filesystem.

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
- **Startup Time Reduction**: Drastically decrease median startup time for Triton Servers fetching model files from external servers
- **Ingress Bandwidth Reduction**: Significantly reduce total ingress bandwidth by using a single inner data fetching node instead of multiple nodes
- **Versioning**: Automatically detect and update local cache when new model file versions are available
- **TRT Models Pre-compilation**: Support pre-compilation of downloaded models for various TensorRT formats to further speed up start-up time

### Future Goals
- **Storage**: Potential for custom storage system leveraging locality and hardware
- **Encryption**: Support for asset encryption for private model files

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
3. Local client cache in Triton servers

### Database Requirements
Minimal information storage:
- Original asset file URL
- Download timestamp
- Last-Modified header
- Content hash value
- Pub-sub channel ID (if downloading)
- Pod ID with in-memory cache

## Architecture

### Components
1. **Service Cluster**
   - Handles caching and downloading of models
   - Implements distributed proxy functionality

2. **Client Library**
   - Embedded within servers
   - Multi-language support
   - Uses UCX for data plane
   - Uses gRPC for control plane

### Distribution
- Distributed as Kubernetes manifest
- Includes deployments and operators for stateless and stateful components

## Alternative Solutions

### Direct Competitors
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
