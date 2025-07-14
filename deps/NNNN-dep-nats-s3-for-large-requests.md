# NATS + S3 for Large Requests

**Status**: Draft 

**Authors**: [@tedzhouhk](https://github.com/tedzhouhk)

**Category**: Architecture 

**Replaces**: [Link of previous proposal if applicable] 

**Replaced By**: [Link of previous proposal if applicable] 

**Sponsor**: [@ryanolson](https://github.com/ryanolson)

**Required Reviewers**: [@ryanolson](https://github.com/ryanolson) [@peabrane](https://github.com/peabrane) [@ishandhananjaya](https://github.com/ishandhananjaya)

**Review Date**: [Date for review]

**Pull Request**: [Link to Pull Request of the Proposal itself]

**Implementation PR / Tracking Issue**: [Link to Pull Request or Tracking Issue for Implementation]

# Summary

Currently, Dynamo Distributed Runtime (DRT) uses NATS to communicate between components. However, NATS is not designed for large payloads. This proposal aim to store large payloads in S3 and use NATS to communicate their metadata between components to enable  routing large requests between dynamo components with minimal overheads.

# Motivation

Currently, Dynamo Distributed Runtime (DRT) uses NATS to communicate between components. NATS is not designed to transfer large payloads and has a hard limit of 16MB message size. While this is sufficient for short and single-prompt requests and we can always chunk the requests if they are larger than 16MB, it introduces significant overheads for larger requests both in implementation and performance. Therefore, we need a better approach to handle large requests with long context or batched prompts that can be a few hundreds MBs.

## Goals

* Support large requests up to hundreds of MBs to a few GBs.

* Achieve minimal latency when routing large requests between dynamo components.

# Proposal

When the frontend receives a curl request, it will first check the size of the request. If it is smaller than a threshold, the NATS stream will have the entire request payload similar to what we have now. If it is larger than the threshold, the frontend will upload the major fields (usually the prompts) to a S3 bucket and replace the raw data with S3 urls in the NATS stream. From then, the S3 urls will be used in NATS when routing this requests within dynamo. For a dynamo component that needs to access this field, it will download the S3 object and get the raw data. When the request is completed, the frontend is responsible for deleting the S3 object. Note that S3 can be either from AWS or from a locally hosted server (i.e., MinIO).

## Phase \<\#\> \<Optional Title\>

**Release Target**: Date

**Effort Estimate**: \<estimate of time and number of engineers to complete the phase\>

**Work Item(s):** \<one or more links to github issues\>

**Supported API / Behavior:**

* \<name and concise description of the API / behavior\>

**Not Supported:**

* \<name and concise description of the API / behavior\>

**Reason Rejected:**

\<bulleted list or pros describing why this option was not used\>

**Notes:**

\<optional: additional comments about this solution\>

