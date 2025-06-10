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
  - Downloads files from the same repositories as the base Dynamo & Triton Server
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
- Utilizes NIXL framework and TCP for client-to-pod communication
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
   - Uses NIXL framework and TCP for data plane
   - Uses gRPC for control plane

### Distribution
- Distributed as Kubernetes manifest
- Includes deployments and operators for stateless and stateful components

## Proposed API

The Client API from the Model Manager Client library is intended to be a drop-in replacement for typical model downloaders such as HuggingFace's downloader.

```
from huggingface_hub import hf_hub_download
import joblib
const HF_TOKEN_ENV_VAR: &str = "HF_TOKEN";

REPO_ID = "YOUR_REPO_ID"
FILENAME = "sklearn_model.joblib"

model = joblib.load(
    hf_hub_download(repo_id=REPO_ID, filename=FILENAME)
)
```

The Model Manager abstracts away the complexity of model provider-specific implementations like huggingface, shielding the end user from the underlying details.

Using the Model Manager Downloader, this process becomes as simple as:

```
from modelmanager_client import modelmanager_download
import joblib

let downloadInfo = DownloadInfo {
    repo_id = "YOUR_REPO_ID",
    filename = "sklearn_model.joblib",
    host = "modelmanager-server",
    provider = "HuggingFace",
};

model = joblib.load
    modelmanager_download(downloadInfo)
);
```

We will begin with HuggingFace for the initial release, with the long-term goal of the model manager being to support a wide range of storage providers — including cloud services like AWS S3 and Azure Storage, as well as custom solutions such as Weka, DDN, and others.

For future releases: Every model manager client will maintain a distributed hash table containing information about which client node holds which model and its associated metadata.
When a new client or node comes online, it will use NIXL's cost API to identify the optimal node for transferring the model weights between the two nodes.

### Proposed Kubernetes Workflow

Model Manager is a Kubernetes application itself. It advertises to the Kubernetes cluster, and any application in there can access it through its name.

For instance, a model manager service can be described using the following Kubernetes service file:

```
apiVersion: v1
kind: Service
metadata:
  name: modelmanager-server
spec:
  selector:
    app: modelmanager-server
  ports:
  - port: 3000
    targetPort: 3000
  type: ClusterIP
```

As an example, here is a deployment file for the model manager’s server itself:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: modelmanager-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: modelmanager-server
  template:
    metadata:
      labels:
        app: modelmanager-server
    spec:
      nodeSelector:
        kubernetes.io/hostname: nnoble-desktop
      containers:
      - name: server
        image: modelmanager-server:latest
        command: ["./modelmanager"]
        ports:
        - containerPort: 3000
        imagePullPolicy: IfNotPresent
        env:
        - name: SERVER_URL
          value: "http://modelmanager-server:3000"
        volumeMounts:
        - name: model-storage
          mountPath: /root
      volumes:
      - name: model-storage
        persistentVolumeClaim:
          claimName: nnoble-pvc
          readOnly: false
```

From there, the applications using the Model Manager client within the cluster can simply access the Model Manager server by specifying the string "model-manager-server" in its configuration.

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