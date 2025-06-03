Problem
When the Triton Server runs, it needs to access model files, organized in repositories. The architecture of the system is backend-agnostic, and the type of files being downloaded is only restricted by which framework backend is used. The current mechanism allows downloading model files from various cloud services, or using the local filesystem.
With model files used in the Triton server becoming larger and larger with time, and with the cluster sizes becoming larger and larger, having every single node in a system trying to fetch their files from the cloud every time means the ingress flow is getting larger and larger, while the latency for the backend startup becomes larger and larger.
We are proposing a distributed proxy system which allows for downloading the files from the same set of repositories as the base Triton Server, caching them locally, and allowing local cluster nodes to download from it instead of going to the external repositories.

Goals
Startup time reduction. The median startup time for a Triton Server that’s currently fetching its model files from external servers should drastically decrease.
Ingress bandwidth reduction. With a large number of nodes using a single inner data fetching node instead of each of them, the total ingress bandwidth in the cluster should drastically decrease. This should also decrease costs iif users don't have to access outside the CSP or if they have models stored in access limited CSP offerings.
Versioning. The system should be able to detect when an updated version of a model file is present, and automatically update the local cache with it.
TRT models pre-compilation. It may be beneficial to have a method to pre-compile the downloaded models for the various TensorRT formats required by the cluster, in order to speed up the start-up time further. This means either pre-compiling the files to the required formats in the model manager directly, or accepting files sent back from the nodes into the model manager for further caching.

Future Goals
Storage. Creating a new distributed storage system is out of scope for the Model Manager itself. However, it might be beneficial to have a custom storage system leveraging locality and hardware.
Encryption. It might be beneficial to support asset encryption for privacy over private model files. However, this is out of scope for this current iteration, and could potentially be added later.

Non-Goals
Optimized routing. Load balancing, network throttling, or other advanced network features are out of scope here. The distributed proxy server is a neighbor of the actual Triton Server cluster, and we should consider speed-of-light bandwidth there. Relying on UCX should alleviate any potential need here.
Security. No additional security should be required nor provided by the Model Manager beyond the currently existing mechanisms of file validation and authentication.

Requirements

Proposal
Distributed storage
As there is no need to reinvent the wheel here, the proposed solution should simply use the Container Storage Interface to allow for any sort of storage system to be used. Any more specific storage system for the purpose of acceleration and locality should be implemented separately using the CSI itself. Note that NVMesh might simply be the answer here.
RWX support will be required here, as any pod in the cluster will require to both read and write to the storage at any time. However, we technically only require a specialization of the RWX paradigm: when creating a new file, pods should use the hash of the contents, meaning if two pods are accidentally downloading the same file, then the atomic move from the temporary download location to the final filename will be idempotent.
Networking
For ideal leveraging of HPC networking, the client to pod communication should utilize the UCX framework for the transfer of model files.
Load balancing
Per design, a proxy server is almost stateless. Either a file is present, isn’t present, or is currently being downloaded by another pod. In the latter case, a simple pub-sub mechanism can be employed for a pod waiting on another one to notify the end of the download process. Due to the thundering herd effect of a Triton cluster coming online, this last point is crucial. 
As such, any form of load balancing is tolerable in front of the pod cluster.
Tiers of caching
The distributed storage should be used for cold caching of the asset files, but per-pod in-memory caching should serve as a secondary tier of faster caching, with a shorter TTL.
Pods should be able to communicate with each other in order to redirect a request to a pod which is known to currently have the requested asset in memory.
Client embedded in the Triton servers should be able to have a single, short lived cache locally as well.
Database
No relational information should be required for the purpose of this work. The only information stored should be:
Original URL of an asset file
Timestamp when it was downloaded
The Last-Modified header from the original file
The hash value of the content of the file, to locate it in the distributed storage.
The optional id of a pub-sub channel currently downloading the file.
The id of a pod which last has it in memory for tiered caching.
Maybe this last one should be a list, to have a secondary tier of  load-balancing across pods which have a file in memory.
Given the above requirements, and given the pub-sub requirement from the load balancing case, Redis fits the bill in terms of features requirements. This makes the only hard requirement in terms of dependencies, everything else being modular.

The rough general algorithm for fetching and forwarding files should be as followed:
downloadFile(url):
    temporaryFilename = downloadFile(url)
    hash = calculateHash(temporaryFilename)
    moveFile(temporaryFilename, baseStorage / hash)
    return hash

getFile(url):
    entry = createIfNotExists(url)
    if entry.justCreated:
        entry.hash = downloadFile(url)
    elseif entry.downloading:
        subscribe(entry.downloading)
        wait()
    if checkUpstreamForNewerFile(entry, url):
        entry.hash = downloadFile(url)
    sendFile(entry.hash)
General architecture
The code should be split in two main parts: a service in the form of a cluster of servers handling the caching and downloading of the models, and a client library to embed within any sort of server which wants to leverage the model manager’s caching service. The client libraries should be implemented and distributed in multiple languages to cater for multiple usage cases.
The data plane to transfer the models will be handled by UCX.
The control plane to issue the download requests will be handled by gRPC.
Distribution
This is an application, which can simply be distributed in the form of a Kubernetes manifest, as this is a set of deployments and operators for both stateless and stateful pieces in the cluster.
Alternate Solutions
Direct competitors:
JuiceFS Distributed Cache
Used by other AI-centric applications, such as BentoML to reduce LLM loading times.
Very generic however, and wouldn’t provide all of the advantages we can by specializing into models, such as version control, model recompilation, leveraging fast transmission hardware, and onioned cached into the nodes.
KServe ModelMesh
The closest there is to what MM would be, as it is an inference-centric, model serving service.
Doesn’t provide hardware-level acceleration, or TensorRT pre-compilation support.
Java.
Amazon SageMaker Multi-Model Caching
Very similar to what MM can provide.
Limited to SageMaker and Amazon products.
Hugging Face Datasets Cache
Fits some of the bill, but not a large-scale, distributed solution.
Adjacent technologies:
https://www.run.ai/blog/accelerating-model-loading-with-run-ai-model-streamer
https://developer.nvidia.com/nim
https://docs.nvidia.com/nim-operator/latest/cache.html 