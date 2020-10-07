---
title: workspace-paths
authors:
  - "@sbwsg"
creation-date: 2020-10-07
last-updated: 2020-10-18
status: proposed
---
# TEP-XXXX: Workspace Paths

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Motivation 1: Declaring Workspace Content](#motivation-1-declaring-workspace-content)
  - [Motivation 2: Improving Tekton's Runtime Validation of Workspaces](#motivation-2-improving-tektons-runtime-validation-of-workspaces)
  - [Motivation 3: Catalog Tasks Already Do This](#motivation-3-catalog-tasks-already-do-this)
  - [Motivation 4: &quot;Woolly&quot; Reasons](#motivation-4-woolly-reasons)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Requirements](#requirements)
- [Proposal](#proposal)
  - [Tasks can declare paths expected on a workspace](#tasks-can-declare-paths-expected-on-a-workspace)
  - [Tasks can declare paths that they'll create on a workspace](#tasks-can-declare-paths-that-theyll-create-on-a-workspace)
  - [TaskRuns can map their own paths to declared Workspace Paths](#taskruns-can-map-their-own-paths-to-declared-workspace-paths)
  - [Tasks and Pipelines use variables to access Workspace Paths](#tasks-and-pipelines-use-variables-to-access-workspace-paths)
  - [Pipelines can declare paths and use them in PipelineTasks](#pipelines-can-declare-paths-and-use-them-in-pipelinetasks)
  - [PipelineRuns can override the paths a Pipeline declares](#pipelineruns-can-override-the-paths-a-pipeline-declares)
  - [Pipelines can thread produced paths from one PipelineTask into expected paths of another](#pipelines-can-thread-produced-paths-from-one-pipelinetask-into-expected-paths-of-another)
  - [TaskRuns and Pipelines can add additional paths a Task has not declared](#taskruns-and-pipelines-can-add-additional-paths-a-task-has-not-declared)
  - [Expected Workspace Paths that do not exist cause a clear error before Steps execute](#expected-workspace-paths-that-do-not-exist-cause-a-clear-error-before-steps-execute)
  - [Produced Workspace Paths that are not created cause a clear error after Steps execute](#produced-workspace-paths-that-are-not-created-cause-a-clear-error-after-steps-execute)
  - [Optional Workspaces that are not provided result in empty variable values](#optional-workspaces-that-are-not-provided-result-in-empty-variable-values)
  - [Extra content on bound Workspaces is totally acceptable](#extra-content-on-bound-workspaces-is-totally-acceptable)
  - [Directories are acceptable Paths](#directories-are-acceptable-paths)
  - [readOnly, ConfigMap and Secret volumes trigger an early error for Tasks that produce Paths](#readonly-configmap-and-secret-volumes-trigger-an-early-error-for-tasks-that-produce-paths)
  - [User Stories](#user-stories)
    - [Clearly declare credential requirements for a git Task](#clearly-declare-credential-requirements-for-a-git-task)
    - [Validate that webpack.config.js Build configuration is passed](#validate-that-webpackconfigjs-build-configuration-is-passed)
    - [Enable arbitrary paths to contain package main](#enable-arbitrary-paths-to-contain-package-main)
    - [Team writing TaskRuns has specific requirements for directory structure of config-as-code](#team-writing-taskruns-has-specific-requirements-for-directory-structure-of-config-as-code)
- [Design Details](#design-details)
- [Test Plan](#test-plan)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
  - [Existing Workaround](#existing-workaround)
  - [Alternative Variable Formats](#alternative-variable-formats)
  - [Alternative Design: Path Params &amp; Results](#alternative-design-path-params--results)
  - [Alternative Implementation: Workspace Files](#alternative-implementation-workspace-files)
- [Upgrade &amp; Migration Strategy](#upgrade--migration-strategy)
- [References](#references)
  - [Related Issue](#related-issue)
  - [Related Designs](#related-designs)
  - [Catalog Examples](#catalog-examples)
<!-- /toc -->

## Summary

Workspace Paths allow a Task author to declare important paths that they expect
to exist on a workspace before or after their Task has run. The paths are validated
by Tekton, providing clear, uniform errors when important files or directories are
missing.

Workspace Paths also allow an author of TaskRuns, Pipelines, and PipelineRuns to
operate on paths expected or produced by Tasks. A TaskRun could override where a
Task looks for an SSH private key, for example, or a Pipeline could name a specific
file that a `git-clone` PipelineTask should checkout and a `docker` PipelineTask
should consume.

## Motivation

This TEP is motivated by several ideas:

* Tasks should be able to declare the content they expect to receive via a Workspace as
well as the content they produce on to a Workspace.
* Pipelines should be able to declare dependencies on specific files or paths from
one PipelineTask to be used by another PipelineTask.
* TaskRuns and PipelineRuns should be able to declare when a path a Task/Pipeline is
expecting lives at a different location on the Workspace that the Run is binding.
They should also be able to override the location that a Task/Pipeline will write
content to on a Workspace.
* Tekton should be better able to validate workspaces at runtime.
* Catalog tasks are already doing this in an ad-hoc way and support should
be made consistently available.

There are also some slightly more "woolly" reasons which I'll summarise at the end
of this section.

### Motivation 1: Declaring Workspace Content

**Tasks**

Task authors currently express whether a workspace is optional as well as where
that workspace will be mounted. Workspace Paths adds an option for Task authors
to declare important files or directories that must either be provided on a
received Workspace or created on a Workspace during execution.

> **Example**:
>
> A `git` Task exposes an `ssh-credentials` workspace that expects the
> following files: `private-key` (with default path `"id_rsa"`) and `known_hosts`
> (default path `"known_hosts"`).
>
> The same `git` Task exposes an `output` workspace that lets the TaskRun
> choose a subdirectory of the Workspace to clone a repo into.

**Pipelines**

Pipeline authors currently express which workspaces are given to which
PipelineTasks. Workspace Paths allows Pipeline authors to define files
that a PipelineTask should produce that other PipelineTasks will consume.

> **Example**:
>
> A `git-clone` PipelineTask checks out a repo and the Pipeline declares that
> this checkout should include a `Dockerfile`. The `Dockerfile` is then fed to
> the `docker-build` Task. The relationship between the two PipelineTasks should
> be understood as a `runAfter` relationship: `git-clone` must run first and
> `docker-build` should run second.

**TaskRuns and PipelineRuns**

TaskRun and PipelineRun authors can currently bind specific volume types to
Workspaces. They should additionally be able to override the expected location
of important files and directories on a Workspace.

> **Example**:
>
> A `git` Task expects a Workspace with a `private-key` path on it, with default
> location of `"/id_rsa"`. A TaskRun can override it to look instead at
> `"/my-keys/id_ecdsa"`.

### Motivation 2: Improving Tekton's Runtime Validation of Workspaces

The only validation Tekton currently performs of Workspaces is against the
Workspace structs in the controller. The Workspace Paths feature would allow for
more precise runtime validation of the paths a workspace has provided or
left out.

> Example: In a `node-webpack-build` Task, a missing `webpack.config.js` file could
> result in a uniform error before the Task's Steps are allowed to run: `Workspace
> path missing: workspace "source-repo" does not provide "webpack_config": expected
> path "/workspace/source-repo/webpack.config.js".`. This would put the TaskRun (or
> PipelineRun) into a clear failed state with the exact Task that expected the file
> along with which file was missing.

### Motivation 3: Catalog Tasks Already Do This

There are several patterns becoming more common in the catalog now that workspaces are
seeing more use:

1. Tasks declare both a `workspace` _as well as_ a `param` for a path into the
workspace. Here's an example from our catalog:

    ```yaml
    workspaces:
    - name: source
      description: A workspace where files will be uploaded from.
    - name: credentials
      description: A secret with a service account key to use as GOOGLE_APPLICATION_CREDENTIALS.
    params:
    - name: path
      description: The path to files or directories relative to the source workspace that you'd like to upload.
      type: string
    - name: serviceAccountPath
      description: The path inside the credentials workspace to the GOOGLE_APPLICATION_CREDENTIALS key file.
      type: string
    ```

    Note above that the `path` and `serviceAccountPath` `params` are providing
    information specific to the `source` and `credentials` workspaces. Two
    things fall out from this:

    - Catalog Task authors clearly see a need to allow the paths into Workspaces to
    be customisable by users of their Tasks.
    - The more workspaces that are needed for a Task, the more params the author
    needs to include to provide this customisable path behavior.

    At time of writing there are 30 catalog entries employing this pattern. See the
    [Catalog Examples](#catalog-examples) reference for links to all of those.

2. Workspace descriptions are required to explain both the purpose of the workspace
_and_ the files that are expected to be received on them. Example from
[recent `tkn` PR](https://github.com/tektoncd/catalog/pull/499/files#diff-6752c482b505c3f8ffed15a7d7d291b5R20-R23):

    > - **kubeconfig**: An [optional workspace](https://github.com/tektoncd/pipeline/blob/master/docs/workspaces.md#using-workspaces-in-tasks)
    > that allows you to provide a `.kube/config` file for `tkn` to access the cluster.
    > The file should be placed at the root of the Workspace with name `kubeconfig`.

### Motivation 4: "Woolly" Reasons

The following items also motivate this TEP but to a lesser degree:

- Decouple declaration of the files and directories a Task needs from their
eventual "location" inside the Task Steps' containers.
- Provide a kind of "structural typing" for the contents of a workspace, and a
way for different but compatible "types" to interface by mapping paths.

### Goals

1. Provide a way for Task authors to declare paths expected on a workspace.
2. Provide a way for Task authors to declare paths that will be produced on a workspace.
3. Provide a way for Pipeline authors to declare important files that one PipelineTask produces and another expects.
4. Allow overriding the paths in TaskRuns and Pipelines/PipelineRuns.
5. Provide clear errors when expected or created paths are missing.
6. Provide a blessed way of declaring paths on a workspace in catalog tasks.
7. Inject new variables that allow Task and Pipeline authors to access paths without hard-coding them.

### Non-Goals

- Declaring any other file features like file types, permissions, content
validation, etc...

## Requirements

* When a workspace is optional, files should only be validated if the workspace
is provided.

* Any variables exposed by Workspace Paths should degrade gracefully if the workspace
is optional and not provided. `$(workspaces.foo.files.bar.path)` should resolve to
an empty string if the optional workspace `foo` is not provided.

* A Workspace Path can be any filesystem entry with a path: a file, a directory,
a symlink, etc.

## Proposal

### Tasks can declare paths expected on a workspace

Workspace declarations in Tasks can include a set of paths:

```yaml
# in a task spec
workspaces:
- name: ssh-credentials
  paths:
    expected:
    - name: private-key
      path: id_rsa
    - name: known_hosts
```

Each entry is given a `name` and optional `path`. If the `path` field is
omitted then the default path of the file is expected to be `"/<name>"`
under the bound workspace's root.

### Tasks can declare paths that they'll create on a workspace

```yaml
# in a task spec
workspaces:
- name: checkout
  paths:
    produced:
    - name: repo
      path: /
```

Again each entry has a `name` and a `path`. The `path` is relative to the
workspace's root. In the example above the `repo` will be checked out
to the root of the workspace.

### TaskRuns can map their own paths to declared Workspace Paths

Workspace bindings can include a set of paths. These are used to map a Task's
declared paths to locations in the Workspace binding's volume.

In the following example, the `private-key` file declared by the Task is
being remapped to the path `/id_ecdsa` rooted in the workspace binding volume.

```yaml
workspaces:
- name: ssh-credentials
  secret:
    secretName: my-ssh-credentials
  paths:
    expected:
    - name: private-key
      path: id_ecdsa
```

In the next example, the TaskRun overrides the _produced_ path for where a
`git-clone` Task will put its code checkout:

```yaml
workspaces:
- name: checkout
  persistentVolumeClaim:
    claimName: my-pvc
  paths:
    produced:
    - name: repo
      path: /src
```

### Tasks and Pipelines use variables to access Workspace Paths

Tasks can use variables with the following structure:

```
workspaces.<workspace-name>.paths.(expected|produced).<path-name>.path
```

Pipelines can use variables with the following structure:

```
workspaces.<workspace-name>.paths.<path-name>.path
```

Pipelines can also use variables to express dependency between PipelineTasks:

```
tasks.<task-name>.workspaces.<workspace-name>.paths.(expected|produced).<path-name>.path
```

These variables all resolve to the absolute path of the file/directory at
runtime, wherever the workspace is mounted and the path is located within
the workspace.

Here's an example that uses the path to a `private-key` file to configure
`ssh` authentication for `git`:

```yaml
kind: Task
spec:
  params:
  - name: git_url
    default: git@github.com:tektoncd/pipeline.git
  workspaces:
  - name: ssh-credentials
    paths:
      expected:
      - name: private-key
        path: /id_rsa
  script: |
    SSH_COMMAND="ssh -i $(workspaces.ssh-credentials.paths.expected.private-key.path)"
    git config core.sshCommand="$SSH_COMMAND"
    git clone $(params.git_url)
```

### Pipelines can declare paths and use them in PipelineTasks

In the example below, the first pipeline task, `clone`, is instructed
to clone the pipelines repo into a `/src` directory on the workspace.
The `build` pipeline task is then told where to look for a webpack
configuration file in the checked out code, under the same `/src`
subdirectory on the workspace.

**Note**: unlike Tasks, Pipelines do _not_ declare a difference between
`expected` and `produced` paths. One PipelineTask may use a Workspace Path
as a `produced` path and another PipelineTask in the Pipeline may use that as
an `expected` path.

```yaml
kind: Pipeline
spec:
  workspaces:
  - name: source
    paths:
    - name: checkout
      path: /
    - name: webpack-config
      path: /webpack.config.js
  tasks:
  - name: clone
    taskRef:
      name: git-clone
    params:
    - name: url
      value: https://github.com/tektoncd/pipeline.git
    workspaces:
    - name: data
      workspace: source
      paths:
        produced:
        - name: checkout-directory
          path: $(workspaces.source.paths.checkout.path)
  - name: build
    taskRef:
      name: webpack-build
    workspaces:
    - name: source-code
      workspace: source
      paths:
        expected:
        - name: webpack-config
          path: $(workspaces.source.paths.webpack-config.path)
```

### PipelineRuns can override the paths a Pipeline declares

PipelineRuns can override all or some of the paths that a Pipeline
Workspace declares. In this example, continuing the example Pipeline above,
the PipelineRun author overrides the location where the webpack-config
file will be found in the checked out source code.

Note: again, no `expected`/`produced` distinction for Pipeline-level workspaces.

```yaml
kind: PipelineRun
spec:
  workspaces:
  - name: source
    persistentVolumeClaim:
      claimName: my-pvc
    paths:
    - name: webpack-config
      path: /frontend/js/webpack.config.js
```

### Pipelines can thread produced paths from one PipelineTask into expected paths of another

Expected Workspace Paths in a PipelineTask can refer to Produced Workspace Paths
from other PipelineTasks. Doing so creates an implicit resource dependency, similarly
to adding a `runAfter` relationship between the two PipelineTasks. See [issue #3109
for this feature request](https://github.com/tektoncd/pipeline/issues/3109).

This also allows us to perform some more validations on a Workspace. We can check
that the Workspace the producer PipelineTask takes is correctly passed to the consumer
PipelineTask. We can also validate that the Workspace Binding is a suitable type for
sharing files in a Pipeline (i.e. it's not an `emptyDir`).

```yaml
kind: Pipeline
spec:
  tasks:
  - name: clone
    taskRef:
      name: git-clone
    workspaces:
    - name: output
      workspace: my-pvc
      paths:
        produced:
        - name: repo-checkout
          path: /src
  - name: build
    taskRef:
      name: webpack-build
    workspaces:
    - name: source
      workspace: my-pvc
      paths:
        expected:
        - name: webpack-config
          path: $(tasks.clone.workspaces.output.paths.produced.repo-checkout.path)/webpack.config.js
```

### TaskRuns and Pipelines can add additional paths a Task has not declared

Workspace bindings can include additional paths that a TaskRun needs
the Task to validate. These can be either `expected` or `produced`
paths.

TaskRuns will use this feature to decide if the Task has failed or not. If
a `git-clone` Task did not clone a repo containing a Dockerfile, for example,
the TaskRun can consider that a failure by adding the following in its workspace
binding:

```yaml
kind: TaskRun
spec:
  workspaces:
  - name: output
    emptyDir: {}
    paths:
      produced:
      - name: repo
        path: /src
      - name: Dockerfile # This was not declared by the Task but _will_ be validated by it
        path: /src/image/Dockerfile
```

Pipelines will use this feature to declare additional paths that can be
threaded through PipelineTasks. Let's look at an example:

```yaml
kind: Pipeline
spec:
  tasks:
  - name: clone
    taskRef:
      name: git-clone
    params:
    - name: url
      value: https://github.com/tektoncd/pipeline.git
    workspaces:
    - name: output
      workspace: my-pvc
      paths:
        produced:
        - name: repo
          path: /src
        - name: Dockerfile
          path: /src/image/Dockerfile
  - name: docker-build
    taskRef:
      name: builder
    params:
    - name: registry
      value: gcr.io
    workspaces:
    - name: source
      workspace: my-pvc
      paths:
        expected:
        - name: Dockerfile
          path: $(tasks.clone.workspaces.output.paths.produced.Dockerfile.path)
```

This will validate that the `clone` Task checked out the files you needed it to
and will report an error before running `docker-build` if it did not.

### Expected Workspace Paths that do not exist cause a clear error before Steps execute

When a TaskRun is executed and any expected Workspace Paths are not present
on the Workspace Binding's volume the TaskRun will be marked as failed
with a `WorkspacePathMissing` reason.

```yaml
status:
  steps:
  - container: step-clone
    imageID: # ...
    name: clone
    terminated:
      reason: WorkspacePathMissing
      message: Workspace Path missing: workspace "config" does not provide
        "private-key" at path "/workspace/config/id_ecdsa"
      containerID: # ...
      exitCode: # ...
      finishedAt: # ...
      startedAt: # ...
```

The TaskRun's log output will include a clear error message explaining which path
is missing:

```bash
Workspace Path missing: workspace "config" does not provide "private-key" at
path "/workspace/config/id_ecdsa".
```

### Produced Workspace Paths that are not created cause a clear error after Steps execute

When a TaskRun is executed but does not create the necessary Produced Workspace
Paths on the Workspace Binding's volume, the TaskRun will be marked as
files with a `WorkspacePathMissing` reason.

```yaml
status:
  steps:
  - container: step-build
    imageID:
    name: clone
    terminated:
      reason: WorkspacePathMissing
      message: Workspace Path missing: workspace "output" does
        not contain "built-executable" that Task was supposed to
        produce at path "/workspace/output/app.exe"
      containerID:
      exitCode:
      finishedAt:
      startedAt:
```

The TaskRun's log output will include a clear error message explaining which path
is missing:

```bash
Workspace Path missing: workspace "output" does not contain "built-executable"
that Task was supposed to produce at path "/workspace/output/app.exe".
```

### Optional Workspaces that are not provided result in empty variable values

When an optional workspace has not been bound by a TaskRun, any variables
referencing that Workspace Path should be replaced with an empty string:

```yaml
workspaces:
- name: config
  optional: true
  paths:
    expected:
    - name: webpack-config
      path: "webpack.config.js"
script: |
  if [ "$(workspaces.config.paths.expected.webpack-config.path)" == "" ]; then
    # user has not supplied the optional config workspace
  fi
```

### Extra content on bound Workspaces is totally acceptable

Just because a Workspace declaration expects specific files does not mean that
the workspace binding can _only_ provide those files. The workspace can contain
as many files as you want but _must_ provide those that are expected by the Task.

In the following example the Secret workspace binding will be mounted with two
files on it. Only one is being explicitly requested by the Task and mapped by
the TaskRun. The Task will receive both, but only one will be validated.
This is totally acceptable:

```yaml
kind: Task
spec:
  workspaces:
  - name: keys
    paths:
      expected:
      - name: private-key
        path: /id_rsa
---
kind: TaskRun
spec:
  workspaces:
  - name: keys
    secret:
      secretName: my-keys
    paths:
      expected:
      - name: private-key
        path: /project-1-ssh-key
---
kind: Secret
apiVersion: v1
metadata:
  name: my-keys
stringData:
  project-1-ssh-key: "L012345=="
  project-2-ssh-key: "L067890=="
```

### Directories are acceptable Paths

"Workspace Paths" are just path mappings. The paths can point to any kind of
filesystem entry including directories. In the following example a "website build"
Task expects to receive a workspace populated with static assets in three
subdirectories: `js/`, `css/`, and `images/`. Just the path names is sufficient
to declare this:

```yaml
kind: Task
spec:
  workspaces:
  - name: static-assets
    paths:
      expected:
      - name: "js"
      - name: "css"
      - name: "images"
```

A TaskRun that provides a Workspace with these three subdirectories would pass
validation successfully:

```yaml
kind: TaskRun
spec:
  workspaces:
  - name: static-assets
    workspace: my-project-assets
```

A TaskRun that builds a project with its own requirements for directory
structure may decide to remap them:

```yaml
kind: TaskRun
spec:
  workspaces:
  - name: static-assets
    workspace: my-project
    paths:
      expected:
      - name: "js"
        path: "/www/minified/_js"
      - name: "css"
        path: "/www/minified/_css"
      _ name: "images"
        path: "/www/img"
```

### readOnly, ConfigMap and Secret volumes trigger an early error for Tasks that produce Paths

Workspaces marked as `readOnly` as well as ConfigMap and Secret volume types trigger
an early error if any Tasks have `produced` paths that would result in attempts to write
on the Workspace.

### User Stories

#### Clearly declare credential requirements for a git Task

As a `git` Task author I want to declare which files my Task expects to appear in the
`ssh-credentials` workspace so that users can quickly learn the necessary files to
provide in their `Secrets`.

#### Validate that webpack.config.js Build configuration is passed

As a `webpack` build Task author I want to validate that `webpack.config.js`
is present in a given workspace before I try to run webpack so that my Task
fails early with a clear error message if an incorrectly populated workspace
has been passed to it.

#### Enable arbitrary paths to contain package main

As the author of a `go-build` Task I want the user to be able to declare a specific
directory to build inside a workspace so that projects with any directory structure
can be built by my Task.

#### Team writing TaskRuns has specific requirements for directory structure of config-as-code

As the author of a TaskRun that builds and releases a project using "config-as-code"
principles I want to organize my config and credential files in my team's repo according
to my org's requirements (and not according to the requirements of the catalog Tasks
I am choosing) so that I am structuring my projects within my company's agreed-upon guidelines.

## Design Details

Workspace Paths will be validated by the `entrypoint` binary. The `entrypoint` will be passed
a list of files and their `expected`/`produced` paths and will look up those paths to confirm
that the files exist. The `entrypoint` to the first Step will receive `expected` files to validate.
The `entrypoint` to the last Step will receive `produced` files to validate.

## Test Plan

- Unit tests for new controller code.
- E2E tests to confirm that Workspace Paths' runtime validation returns expected errors.
- Examples to show correct usage of the Workspace Paths feature.
- Documentation to describe the feature and explain how it's used.

## Drawbacks

- If not documented correctly users could fall in to a trap of trying to declare
every single file or directory their Task produces.

    This wouldn't be ideal because it could result in less generic Tasks. As just one
    example it's conceivable a user could misunderstand this feature and believe they
    need to start writing multiple Tasks: `git-clone-a-docker-file`,
    `git-clone-a-node-project`, `git-clone-a-go-project`, etc etc.

    The best approach for Task authors is going to be to keep the paths they declare
    as broad as possible while TaskRun and Pipeline authors become more selective
    in the specific paths they want validated.

## Alternatives

### Existing Workaround

1. Expose a `param` for each file path that TaksRun authors can customize.
2. Manually validate that each file you are interested in has been provided in your `script`
or `command`. E.g. in bash:

```bash
if [ -f "$(workspaces.foo.path)/$(params.path)" ]; then
```

### Alternative Variable Formats

There are variations on the variable formats proposed so far that
could shorten them. We could drop the trailing `.path`, or omit
the `.(expected|produced).` portion.

**Pro**:
- Shorter

**Con**:
- No longer follow the nesting of the JSON/YAML

### Alternative Design: Path Params & Results

1. Allow Tasks and TaskRuns to declare "Path Params":
    ```yaml
    kind: Task
    spec:
      params:
      - name: private-key
        type: path
        default: $(workspaces.ssh-credentials)/id_rsa
    ---
    kind: TaskRun
    spec:
      params:
      - name: private-key
        value: $(workspaces.ssh-credentials)/id_ecdsa
    ```

2. Allow Tasks to declare "Path Results" as well

    ```yaml
    kind: Task
    spec:
      results:
      - name: compiled-binary
        type: path
        value: "$(workspaces.output.path)/app.exe"
    ```

3. Allow these to be linked in a Pipeline:

    ```yaml
    kind: Pipeline
    spec:
      tasks:
      - name: build
        image: build-binary:latest
        workspaces:
        - name: output
          workspace: shared-pvc
      - name: upload
        image: push-to-bucket:latest
        params:
        - name: path-to-upload
          value: $(tasks.build.results.compiled-binary)
    ```

**Pros**:
- Leverage existing dependency resolution between Params and
Results to infer `runAfter` for Workspace usage.
- We might be able to infer which Workspaces are passed to
which PipelineTasks using this method.

**Cons**:
- Adds a "mode" to params/results: originally they exist only
as name/value. This would overload them to be name/value/file-content.
- This would overload params in another way: the validation of
the paths performed by the entrypoint would be an extra "feature"
of path params that user would need to understand.
- For Tasks with multiple workspace declarations it might be
difficult to infer exactly which workspace declaration should
be bound with a workspace that a PipelineTask param is linking.

### Alternative Implementation: Workspace Files

`Workspace Files` is a slimmed-down version of `Workspace Paths` that only provides
for `expected` files. The result is a thinner syntax:

```yaml
kind: Task
spec:
  workspaces:
  - name: credentials
    files:
    - name: private-key
      path: /id_rsa
---
kind: TaskRun
spec:
  workspaces:
  - name: credentials
    secret:
      name: my_keys
    files:
    - name: private-key
      path: /id_ecdsa
```

This approach was taken in the initial draft of the feature. The utility is
limited, however: without a disinction between `expected` and `produced`
files there's no way to declare things like a checkout directory for a
`git` repo. Why? Because the checkout directory won't exist before the Task
runs, so validation checking the existence of the checkout directory would
fail. This limits the feature to input paths only - those that will exist
when the Task executes.

## Upgrade & Migration Strategy

Workspace Paths should be entirely backwards-compatible. A workspace declaration that does
not include them will not perform any path validation when the TaskRun executes.

## References

### Related Issue

- Some of the features stem from requirements in the issue [Improve UX of getting credentials into Tasks](https://github.com/tektoncd/pipeline/issues/2343).

### Related Designs

- Echoes some of the described features of the [FileSet Resource](https://docs.google.com/document/d/1euQ_gDTe_dQcVeX4oypODGIQCAkUaMYQH5h7SaeFs44/edit#bookmark=id.qblcy95l5zsk) in the PipelineResources redesign from winter 2019/20.
- File contract features of the [PipelineResources revamp](https://docs.google.com/document/d/1KpVyWi-etX00J3hIz_9HlbaNNEyuzP6S986Wjhl3ZnA/edit#heading=h.8e6t5h2q2zt2).

### Catalog Examples

Here are all of the examples in the catalog where we ask for both a workspace and
a param providing a path into that workspace.

1. [02-build-runtime-with-gradle](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/pipeline/openwhisk/0.1/tasks/java/02-build-runtime-with-gradle.yaml)
1. [03-build-shared-class-cache](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/pipeline/openwhisk/0.1/tasks/java/03-build-shared-class-cache.yaml)
1. [04-finalize-runtime-with-function](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/pipeline/openwhisk/0.1/tasks/java/04-finalize-runtime-with-function.yaml)
1. [01-install-deps](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/pipeline/openwhisk/0.1/tasks/javascript/01-install-deps.yaml)
1. [02-build-archive](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/pipeline/openwhisk/0.1/tasks/javascript/02-build-archive.yaml)
1. [03-openwhisk](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/pipeline/openwhisk/0.1/tasks/javascript/03-openwhisk.yaml)
1. [01-install-deps](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/pipeline/openwhisk/0.1/tasks/python/01-install-deps.yaml)
1. [02-build-archive](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/pipeline/openwhisk/0.1/tasks/python/02-build-archive.yaml)
1. [03-openwhisk](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/pipeline/openwhisk/0.1/tasks/python/03-openwhisk.yaml)
1. [ansible-runner](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/task/ansible-runner/0.1/ansible-runner.yaml)
1. [build-push-gke-deploy](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/task/build-push-gke-deploy/0.1/build-push-gke-deploy.yaml)
1. [buildpacks-phases](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/task/buildpacks-phases/0.1/buildpacks-phases.yaml)
1. [buildpacks](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/task/buildpacks/0.1/buildpacks.yaml)
1. [create-github-release](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/task/create-github-release/0.1/create-github-release.yaml)
1. [gcs-create-bucket](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/task/gcs-create-bucket/0.1/gcs-create-bucket.yaml)
1. [gcs-delete-bucket](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/task/gcs-delete-bucket/0.1/gcs-delete-bucket.yaml)
1. [gcs-download](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/task/gcs-download/0.1/gcs-download.yaml)
1. [gcs-generic](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/task/gcs-generic/0.1/gcs-generic.yaml)
1. [gcs-upload](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/task/gcs-upload/0.1/gcs-upload.yaml)
1. [git-batch-merge](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/task/git-batch-merge/0.1/git-batch-merge.yaml)
1. [git-batch-merge](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/task/git-batch-merge/0.2/git-batch-merge.yaml)
1. [git-clone](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/task/git-clone/0.1/git-clone.yaml)
1. [git-clone](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/task/git-clone/0.2/git-clone.yaml)
1. [github-app-token](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/task/github-app-token/0.1/github-app-token.yaml)
1. [gke-cluster-create](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/task/gke-cluster-create/0.1/gke-cluster-create.yaml)
1. [jib-gradle](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/task/jib-gradle/0.1/jib-gradle.yaml)
1. [jib-maven](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/task/jib-maven/0.1/jib-maven.yaml)
1. [kaniko](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/task/kaniko/0.1/kaniko.yaml)
1. [maven](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/task/maven/0.2/maven.yaml)
1. [wget](https://github.com/tektoncd/catalog/blob/972bca5c5642a056c28ff1976c30c7367e6a9c3a/task/wget/0.1/wget.yaml)
