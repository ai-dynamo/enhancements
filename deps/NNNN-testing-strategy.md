# Test Strategy

**Status**: Draft

**Authors**: nnshah1, harrison

**Category**: Architecture 

**Replaces**: N/A

**Replaced By**: N/A

**Implementation**: N/A

**Sponsor**: nnshah1

**Required Reviewers**: TBD

**Review Date**: TBD

**Pull Request**: TBD

**Issue(s)**: NA

# Summary

Proposes a set of development and release life-cycle test stages, test
types, coverage areas, test environments and test runners for
dynamo. These will be used to form actual test plans.

# Motivation

Currently the dynamo project has a number of different test strategies
and implementations which can be confusing in particular with respect
to what tests run, when, and where. There is not a guide for
developers, QA or operations teams as to the general theory and basic
set of tools, tests, or when and how they should be run. We need a set
of guidelines and overarching structure to help form the basis for
test plans.

## Goals

* Provide a common set of terms to ease communication about test plans and results

* Tests should be easy to write and run both locally and for CI

* Tests should cover the entire development and release life-cycle 

* Test should be automated 

* Tests should cover code, features, and documentation and have a report of coverage.

* Strategy must fit with the polyglot nature of dynamo (support for
  multiple programming languages and deployment targets)

### Non Goals

* To enumerate all tests and test cases (test plan)


## Requirements

> TBD, May remove

**\[Optional \- if not applicable omit\]**

List out any additional requirements in numbered subheadings.

**\<numbered subheadings\>**

### REQ \<\#\> \<Title\>

Describe the requirement in as much detail as necessary for others to understand it and how it applies to the TEP. Keep in mind that requirements should be measurable and will be used to determine if a TEP has been successfully implemented or not.

Requirement names should be prefixed using a monotonically increasing number such as “REQ 1 \<Title\>” followed by “REQ 2 \<Title\>” and so on. Use title casing when naming requirements. Requirement names should be as descriptive as possible while remaining as terse as possible.

Use all-caps, bolded terms like **MUST** and **SHOULD** when describing each requirement. See [RFC-2119](https://datatracker.ietf.org/doc/html/rfc2119) for additional information.

# Proposal

The testing strategy will be a living document stored in the
repository detailing the decisions and definitions outlined here. This
proposal will serve as a starting point and enumerate the sections of
the living testing strategy.


## Dynamo Testing Strategy (Intro)

Dynamo is a distributed inference serving framework designed for
generative AI use cases. Like any project it has a development and
release life-cycle with different testing requirements, test types that
have a different goal and modules and functionality that require test
coverage.

## Development and Release Life-Cycle Test Stages

At each stage of development and release there are a different set of
tests that are required. To help organize this we will group tests
into categories and label them to clearly indicate where and when they
run and what part of the life-cycle they "gate". Tests can be in more
than one category.

### Pre-Commit

Pre-commit tests and gates are controlled and exercised using the [pre-commit framework](https://pre-commit.com/).

Configuration for dynamo is at the top level [pre-commit-config.yaml](https://github.com/ai-dynamo/dynamo/blob/main/.pre-commit-config.yaml).

Tests that are part of pre-commit are:

1. Code formatting
2. Spelling
3. Basic Code linting. Linting that is fast and doesn't depend on
   project dependencies.

These must include sections for all languages used in dynamo including rust and python.

Coverage: all files 

Exception: 

* auto generated files

* foreign files (files from other projects used without modification)

> Consider a way to audit exceptions 

### Pre-Merge

Pre-merge tests are required to pass before code can be merged to
main. These tests are designed to be a set of sanity and core
functionality tests that have broad coverage and give confidence that
a change hasn't broken core functionality.


### Nightly

### Weekly

### QA

### Pre-Release

## Test Types

### Unit

### Integration

### End to End

### Performance

### COTs

## Test Coverage

### Feature Coverage

### Code Coverage

### Tutorials / Examples

## Test Runners

## Test Environments


# Implementation Details

-------------------------------------------------------------------------------------------
| Life-cycle | Test Runner | Local Environment | Local Command | CI Environment | CI Command | 
------------------------------------------------------------------------------------------- 
| pre-commit | [pre-commit framework](https://pre-commit.com/). | Host | `pre-commit run --all` | github action | [pre-merge.yml](https://github.com/ai-dynamo/dynamo/blob/main/.github/workflows/pre-merge.yml) |


> TBD

**\[Optional \- if not applicable omit\]**

Add additional detailed items here including interface signatures, etc. Add anything that is relevant but seems more of a detail than central to the proposal. Use sub sections / bullet points as needed. Try to provide images and diagrams to facilitate understanding. If applicable link to PR.

## Deferred to Implementation

> TBD

**\[Optional \- if not applicable omit\]**

List out items that are under discussion but that will be resolved only during implementation / code review. 

# Implementation Phases


## Phase 1 Baseline

**Release Target**: Date

**Effort Estimate**: \<estimate of time and number of engineers to complete the phase\>

**Work Item(s):** \<one or more links to work items, usually to JIRA tasks or user stories\>

**Supported API / Behavior:**

* Ensure formatting and linting is in pre-commit for all languages


**Not Supported:**

* \<name and concise description of the API / behavior\>

# Related Proposals

**\[Optional \- if not applicable omit\]**

* File

* File

* File

* File

* File

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

