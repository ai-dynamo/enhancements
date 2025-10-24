# LoRA Support for Dynamo

This document outlines the support for LoRA models in Dynamo.

## Overview

Each LoRA Model will have a unique identifier and a corresponding `Model Deployment Card` in Dynamo with additional lora specific metadata (max_rank, module_names, etc.). We can use most of the code path (routing, preprocessing, etc.) for base model for LoRA models in frontend.

Single backend worker can serve multiple LoRA models. Each worker will have a LoRA manager to manage the lifecycle of the LoRA models locally.


## LoRA Model

Interfaces and dataclasses for LoRA model in Dynamo. Corresponding rust structs and traits will be available in `dynamo/lib/llm/src/lora.rs`.

Data structures for LoRA model management in Dynamo:

```python
from dataclasses import dataclass

# LoRA model dataclass in Dynamo
@dataclass
class LoraModel:
    name: str
    status: LoraStatus
    cache_location: CacheLocation
    local_path: str

class LoraStatus(intEnum):
    DOWNLOADED = 1
    LOADED = 2
    READY_FOR_USE = 3

class CacheLocation(Enum):
    DEVICE = "device"
    HOST = "host"
    DISK = "disk"
    REMOTE = "remote"

# LoRA Events
class LoraEventType(intEnum):
    DOWNLOADED_TO_DISK = 1
    DELETED_FROM_DISK = 2
    LOADED_TO_DEVICE = 3
    UNLOADED_FROM_DEVICE = 4

@dataclass
class LoraEvent:
    event_type: LoraEventType
    lora_model: LoraModel
    timestamp: datetime
    instance_id: str

# LoRA Manager - this will be responsible for managing lifecycle of the LoRA models in backend service
class LoraManager(abc.ABC):
    @abc.abstractmethod
    def lora_models(self) -> List[LoraModel]:
        pass

    @abc.abstractmethod
    def load_lora(self, name: str) -> LoraModel:
        pass

    @abc.abstractmethod
    def unload_lora(self, name: str) -> LoraModel:
        pass

# Concrete implementation of LoRA manager
class DynamoLoraManager(LoraManager)
    pass

# Plugin interface for LoRA downloader in LLM backend
class LoraDownloaderBase(abc.ABC):
    @abc.abstractmethod
    def download_lora_to_disk(self, name: str) -> LoraModel:
        pass

    @abc.abstractmethod
    def delete_lora_from_disk(self, name: str) -> LoraModel:
        pass

# Dynamo's default concrete LoRA downloader implementation
# this will downlowd LoRA models from huggingface to local disk 
class HuggingFaceDownloader(LoraDownloaderBase):
    ...

```

## LLM Backends 

LLM backend frameworks will be responsible for:
- Downloading LoRA models from upstream LoRA repository to local disk
- Loading LoRA models to device memory
- Unloading LoRA models from device memory
- Deleting LoRA models from local disk

Support custom LoRA downloader plugin to download LoRA models to local disk.
```
# Default LoRA downloader is HuggingFaceDownloader
python -m dynamo.vllm  \
    --lora-downloader-plugin="dynamo.lora.downloader.HuggingFaceDownloader"

# Users can bring their own Custom LoRA download logic
python -m dynamo.vllm  \
    --lora-downloader-plugin="my-org.lora.downloader.CustomDownloader"
```

## Dataflow for LoRA model requests

When a request comes in, Frontend will identify it as a LoRA request and set `is_lora` flag to True in dynamo pre-procesed request object.

Backend can 
1. reuse existing chat/completion endpoint for LoRA models (Preffered)
2. or use dedicated LoRA specific chat/completion endpoints for LoRA models

In Backend layer we can convert the pre-processed request object to LoRA specific request object for the framework (vllm, sglang, tensorrt-llm, etc.)

## KV routing with LoRA models

LoRA enabled Router (in Frontend) will discover all the LoRA models using service discovery mechanism. 
Each active LoRA model will have dedicated KV router to handle the request for the LoRA model. It will function similar to existing KV router for base model.

Backend requests will generate KV event stream and Frontend router will consume them to build its Radix Tree.

## Framework Support for multiple concurrent LoRA model serving

### vLLM LoRA Support
todo: need to ensure that vLLM is built with punica kernels support for serving multiple LoRA models concurrently.

### sglang LoRA Support

### TensorRT-LLM LoRA Support
