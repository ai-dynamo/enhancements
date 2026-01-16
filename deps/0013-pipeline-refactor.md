# <Modular Pipeline Architecture for Dynamo Frontend>

**Status**: Draft 

**Authors**: [Name/Team] 

**Category**: Architecture

**Replaces**: Current build_routed_pipeline_with_preprocessor implementation in lib/llm/src/entrypoint/input/common.rs 

**Replaced By**: [Link of previous proposal if applicable] 

**Sponsor**: [Name of code owner or maintainer to shepard process]

**Required Reviewers**: [Names of technical leads that are required for acceptance]

**Review Date**: [Date for review]

**Pull Request**: [Link to Pull Request of the Proposal itself]

**Implementation PR / Tracking Issue**: [Link to Pull Request or Tracking Issue for Implementation]

# Summary

This proposal calls for the option of using a third party router in Dynamo. The first use case is EPP. 

# Motivation

The current `build_routed_pipeline_with_preprocessor` function in `lib/llm/src/entrypoint/input/common.rs` conflates routing selection with request processing when used with a 3rd party router (i.e. GAIE EPP).
The decision to route to a specific worker is controlled through backend_instance_id in annotations/routing hints.
When GAIE EPP determines routing it calls this pipeline with the `query_instance_id` annotation and the pipeline determines the workers and short-circutes without serving the request.
When GAIE serves this request this pipeline is called again. The router looks for the hints and if present, routes there instead of figuring out the workers. The presence of these hints is used as a signal not to figure out the routing again.

In the new approach the routing is figured out by EPP calling the Prefill_Router directly through new bindings and the pipeline is only instantiated once during the request serving. All of the book-keeping operations will also be called on the Prefill_Router instance.

During GAIE request serving the FrontEnd has to be instantiated with the `--direct-route` cli flag which tells the pipeline to route directly. It translates into the existing RouterMode::Direct() and the router expects for the hints to be in the body. It is the 3rd party router's responsibility to provide them. Dynamo will error out if they are not provided. 
In this mode the router will NOT be doing book keeping and no router events coordination will be needed.


## Goals

* Allow for the 3rd party Router.
* Allow for the PrefillRouter to be called outside of the pipeline.
* Do not do book keeping in the Direct routing mode.
* Remove the router sync flag from the FrontEnd in case of the direct rotuing 


## Requirements

### REQ 1 Allow for the 3rd party Router
The first use case is the GAIE EPP.

### REQ 2 Allow for the Router to be called outside of the p pipeline
When EPP needs to know the worker(s) it does not need the entire pipeline. We can expose the PrefillRouter as a stand-alone.

### REQ 3 Routing Hints enhancement
Allow for the routing hints to be read from the headers and if not found then from the nvext annotation in the body.
The nvext values also have to be preserved because the  NAT (Nemo Agentic Toolkit) team uses them. TBD: if the NAT team does not use the `query_instance_id` annotation to request the worker ids only instead of serving, we need to remove it to streamline the logic.

# Proposal

See this [PR] (https://github.com/ai-dynamo/dynamo/pull/5446) for proposed changes.

# Related Proposal
- Clients who want to handle their own preprocessing / post processing in their modules or rely on Inference Engines. See [frontend: Proposal for having Python orchestrate the request handling](https://github.com/ai-dynamo/enhancements/pull/52/changes)
When an alternative python pipeline is implemented it should also support Direct Routing.
The bindings with pre-processing needs to be able to also call Vllm.
A possibility is that we will create a python PrefillRouter binding and instantiate it as a service for EPP. 
This python bonding will be needed for the primary serving pipeline anyway. This is TBD.