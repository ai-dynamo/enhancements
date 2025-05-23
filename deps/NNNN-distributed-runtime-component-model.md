# Distributed Runtime Component Model

**Status**: Draft 

**Authors**: graham, nnshah1, biswa, ishan, maksim

**Category**: Architecture 

**Replaces**: [Link of previous proposal if applicable] 

**Replaced By**: [Link of previous proposal if applicable] 

**Sponsor**: [Name of code owner or maintainer to shepard process]

**Required Reviewers**: [Names of technical leads that are required for acceptance]

**Review Date**: [Date for review]

**Pull Request**: [Link to Pull Request of the Proposal itself]

**Implementation PR / Tracking Issue**: [Link to Pull Request or Tracking Issue for Implementation]

# Summary


# Motivation

The definition of a `component` and its relationship to the `runtime`
is not always clear as we move between `python` and `rust`. In
different parts of the documentation and code base similar terms such
as `client`, `worker`, `component`, `service` are used
interchangeabley which can be confusing.

We'll clarify the architecture and document the decsisions here to
help clarify and discuss any changes needed. In cases where no change
is neededd this will serve as a "retro-active" proposal.


## Goals

* Clearly define a `component`, its properties, and provide examples of `rust` and `python` components. 

* Prescriptively define how components use the `runtime` to discover and access each other.

* Define how components are discovered and configured

* Define how components are identified - including namespaces

* Define how endpoints are hosted and used


### Non Goals

* Radically reset current terms and definitions if not needed



## Requirements

### REQ \<\#\> \<Title\>

TBD

# Proposal

## Namespace

## Component

## Endpoint

## Event

## Property

## Runtime

## Request Flow

## Discovery Flow


# Alternate Solutions

**\[Required, if not applicable write N/A\]**

List out solutions that were considered but ultimately rejected. Consider free form \- but a possible format shown below.

## Alt \<\#\> \<Title\>

**Pros:**

\<bulleted list or pros describing the positive aspects of this solution\>

**Cons:**

\<bulleted list or pros describing the negative aspects of this solution\>

**Reason Rejected:**

\<bulleted list or pros describing why this option was not used\>

**Notes:**

\<optional: additional comments about this solution\>

