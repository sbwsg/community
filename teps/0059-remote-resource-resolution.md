---
status: proposed
title: Remote Resource Resolution
creation-date: '2021-03-23'
last-updated: '2021-03-23'
authors:
- '@sbwsg'
---

# TEP-0059: Remote Resource Resolution

<!-- toc -->
- [Summary](#summary)
- [Key Terms](#key-terms)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
  - [Use Cases (optional)](#use-cases-optional)
- [Requirements](#requirements)
- [Proposal](#proposal)
  - [Notes/Caveats (optional)](#notescaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
  - [User Experience (optional)](#user-experience-optional)
  - [Performance (optional)](#performance-optional)
- [Design Details](#design-details)
- [Test Plan](#test-plan)
- [Design Evaluation](#design-evaluation)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
  - [The User-Facing Syntax](#the-user-facing-syntax)
- [Pros:](#pros)
- [Cons:](#cons)
  - [The Protocol](#the-protocol)
- [Infrastructure Needed (optional)](#infrastructure-needed-optional)
- [Upgrade &amp; Migration Strategy (optional)](#upgrade--migration-strategy-optional)
- [Future Extensions](#future-extensions)
- [References (optional)](#references-optional)
<!-- /toc -->

## Summary

This TEP builds upon the ideas and existing implementation of Pipelines' OCI
Tekton Bundles support. Today Pipelines has two places that Tasks and Pipelines
can live and be used from: the cluster that Pipelines is deployed in and OCI
registries that contain Tekton Bundles (when feature flag enabled).

## Key Terms

This section defines some key terms used throughout the rest of this doc.

- "Remote": Any location that stores Tekton resources outside of the cluster
  that Pipelines is running in.  Could be: oci registries, git repos, other
  namespaces than the one the user's pipelines run in, other clusters, cloud
  buckets, etc.
- "Remote Resource": The YAML file or other document that lives in the Remote.
- "Resolver": A program or piece of code that knows how to interpret a
  reference to a remote resource and fetch it
- "Resource Resolution": The act of taking a reference to a resource in a
  Remote, fetching it, and using it as part of a TaskRun/PipelineRun.

## Motivation

Change Pipelines' role in resource resolution from that of direct resolver to
that of protocol enforcer.

Problems with existing approach:

1. Users have to manually go explore the catalog and discover tasks, download
   them, apply them, etc...
2. Tutorials have to include additional steps to account for the `kubectl apply
   task.yaml` actions
3. The level of control afforded to Operators by this is extremely narrow:
   Either a resource exists in the cluster or it doesnt.  To add new Tasks and
   Pipelines Operators have to manually (or via automation) sync them into the
   cluster (aka `kubectl apply -f task.yaml`)--

### Goals

- Decouple Pipelines from responsibility of resource resolution.
    - Change Pipelines' role to offering a protocol instead of resolving
      resources by itself.
- Agnostic to Pipeline types: Support Tasks, Pipelines, Custom Tasks and any
  other resources that Tekton Pipelines might choose to add support for (e.g.
  Steps).
    - Note: this doesn't mean that we'd add support for "any" resource type in
      the initial implementation. We may decide to support only fetching Tasks
      initially, for example. But the protocol itself should be indifferent to
      the content being fetched.
- Enable Operators to decide which remotes can be used as a source of new Tasks
  in their clusters.
    - And therefore provide the ability to manage Tasks in an org's source of
      choice - pushing new Tasks by `git push`, `gsutil upload`, etc etc,
      instead of `kubectl apply`.
- Asynchronous resolution that doesn't block a reconcile thread while it is
  happening.
- Backwards compatibility with the existing flagged Tekton Bundles support.
    - API surface the same but switch to async resolution.
- Per-source RBAC
- Allow Pipelines and downstreams to choose which sources are enabled OOTB with
  a release
- Support for credentials both per-source or per-workload depending on the
  requirements of the source.  (e.g. operators configures git creds once or, if
  allowed by operator, users can supply creds with each invocation
  (multi-tenant))
- Separate configurable timeout for async resolution so operators can tune the
  allowable delay before remote resolution is considered failed
    - Also the Events, Conditions, Statuses to support communicating these
      failures
    - Would this ^ be enough for building reports of how quick tasks are being
      resolved? If not, do whatever is needed to make this easier too.
- An agreed-upon syntax that platforms and tools can use to reason about the
  remote locations that Tasks need to be fetched from

### Non-Goals

<!--
What is out of scope for this TEP?  Listing non-goals helps to focus discussion
and make progress.
-->

### Use Cases (optional)

<!--
Describe the concrete improvement specific groups of users will see if the
Motivations in this doc result in a fix or feature.

Consider both the user's role (are they a Task author? Catalog Task user?
Cluster Admin? etc...) and experience (what workflows or actions are enhanced
if this problem is solved?).
-->

## Requirements

<!--
Describe constraints on the solution that must be met. Examples might include
performance characteristics that must be met, specific edge cases that must
be handled, or user scenarios that will be affected and must be accomodated.
-->

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation.  The "Design Details" section below is for the real
nitty-gritty.
-->

### Notes/Caveats (optional)

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above.
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->

### Risks and Mitigations

<!--
What are the risks of this proposal and how do we mitigate. Think broadly.
For example, consider both security and how this will impact the larger
kubernetes ecosystem.

How will security be reviewed and by whom?

How will UX be reviewed and by whom?

Consider including folks that also work outside the WGs or subproject.
-->

### User Experience (optional)

<!--
Consideration about the user experience. Depending on the area of change,
users may be task and pipeline editors, they may trigger task and pipeline
runs or they may be responsible for monitoring the execution of runs,
via CLI, dashboard or a monitoring system.

Consider including folks that also work on CLI and dashboard.
-->

### Performance (optional)

<!--
Consideration about performance.
What impact does this change have on the start-up time and execution time
of task and pipeline runs? What impact does it have on the resource footprint
of Tekton controllers as well as task and pipeline runs?

Consider which use cases are impacted by this change and what are their
performance requirements.
-->

## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable.  This may include API specs (though not always
required) or even code snippets.  If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.

If it's helpful to include workflow diagrams or any other related images,
add them under "/teps/images/". It's upto the TEP author to choose the name
of the file, but general guidance is to include at least TEP number in the
file name, for example, "/teps/images/NNNN-workflow.jpg".
-->

## Test Plan

<!--
**Note:** *Not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy.  Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations).
-->

## Design Evaluation
<!--
How does this proposal affect the reusability, simplicity, flexibility 
and conformance of Tekton, as described in [design principles](https://github.com/tektoncd/community/blob/master/design-principles.md)
-->

## Drawbacks

<!--
Why should this TEP _not_ be implemented?
-->

## Alternatives

This section describes some ways this TEP could be implemented. These are
intended initially to spur discussion with the idea being we'll gravitate
towards and eventually pick the most suitable approach. We can add ideas to
this list as they surface.

The possibilities are broken down into three sections: User-facing syntaxes,
protocols that pipelines could support, ways that resolvers could operate.

### The User-Facing Syntax

1. Custom Task protocol w/ `apiVersion`, `bundle` and `name` as keys.

```yaml
taskRef:
  apiVersion: gitref.tekton.dev/v1alpha1
  bundle: git@github.com:foo/foo.git
  name: /path/to/yaml.yaml
```

Pros:
- 
Cons:
- 

2. Typed entries in the taskRef (originally proposed in [TEP-0005's Alternatives](#TODOLINK)):

```yaml
taskRef:
  name: foo
  git:
    repo: git@github.com:foo/foo.git
    path: /path/to/yaml.yaml
```

3. Add a `remote` raw field to taskref that accepts any remote config:

```yaml
taskRef:
  name: foo
  remote:
    type: git
    repo: git@github.com:foo/foo.git
    path: /path/to/yaml.yaml
```

```yaml
taskRef:
  name: bar
  remote:
    type: gcs
    bucket: gs://my-tekton-tasks/
    path: /path/to/yaml.yaml
```

### The Protocol

1. Custom Tasks that apply resources to the cluster directly.
  - This is how the CatalogTask PR operates [insert link here](#TODOLINK).

2. Controllers that return YAML via an agreed-upon protocol in Runs.
  - This is how the initial proof-of-concept operates [insert link here](#TODOLINK).

3. Introduce a new CRD type (`ResourceResolution`?)

3. Admission / mutating webhook where resolution happens as the document hits
the k8s API server.
    - See Billy's comments on speedy resolution via the Github repository
      content API

4. Service-driven approach using webhooks and label selectors, also returning
YAML via an agreed-upon protocol.

## Infrastructure Needed (optional)

<!--
Use this section if you need things from the project/SIG.  Examples include a
new subproject, repos requested, github details.  Listing these here allows a
SIG to get the process for these resources started right away.
-->

## Upgrade & Migration Strategy (optional)

- Must be syntax-compatible with the existing Tekton Bundles feature or
  must otherwise operate comfortably side-by-side with it.

## Future Extensions

- Setting "default" remotes so that operators can pick where to get tasks from
  by default (e.g. internal oci registry might be the default for some orgs.
  "git-clone" by itself would automatically result in a fetch from that oci
  registry)
- Building a reusable library that Resolvers can leverage so they don't have to
  each rewrite their side of the protocol, common caching approaches, etc.
- Replacing ClusterTask with a namespace full of resources that only a Resolver
  has access to.


## References (optional)

- [TEP-0005: Tekton OCI Bundles](#TODOLINK)
    - Alternatives section describes possible syntax for git refs
- [Tekton Bundles docs in Pipelines](#TODOLINK)
- [Tekton Bundle Contract](#TODOLINK)
- [Tekton OCI Images Design](#TODOLINK)
- [Tekton OCI Image Catalog](#TODOLINK)
    - Future Work section describes expanding support to other task locations
- [Referencing Tasks in Pipelines: Separate Authoring and Runtime Concerns](#TODOLINK)
- [Use tasks from Git issue in pipelines](#TODOLINK)
- [CatalogTask Custom Task PR + associated discussion](#TODOLINK)
- [Pipelines Discussion about running tasks from the catalog](#TODOLINK)
- [Issue 3305: Refactor the way resources are injected into the reconcilers](https://github.com/tektoncd/pipeline/issues/3305)
