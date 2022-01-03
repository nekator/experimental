# commit-status-tracker Go Operator

## Overview

This operator tracks completed [Tekton](https://github.com/tektoncd/pipeline) [PipelineRuns](https://github.com/tektoncd/pipeline/blob/master/docs/pipelineruns.md) and attempts to create a [GitHub Commit Status](https://developer.github.com/v3/repos/statuses/) with the success or failure of the PipelineRun.

## Why?

If you're running tasks that are important parts of your deployment flow, you
can define policies that require specific checks are carried out before code can
be merged.

These can be enforced by GitHub, using their [branch protection](https://help.github.com/en/github/administering-a-repository/configuring-protected-branches) mechanism.

If you want your Tekton Pipelines to be a part of this, then you'll want to report the success or failure of your PipelineRuns to Github (you might also want Tasks, but that's not implemented yet).

This is an [operator-sdk](https://github.com/operator-framework/operator-sdk) originated operator.

## Annotating a PipelineRun

The operator watches for PipelineRuns with specific annotations.

This is an alpha operator, and the annotation names will likely change, but for now
you'll need...

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineRun
metadata:
  name: demo-pipeline-run
  annotations:
    "tekton.dev/git-status": "true"
    "tekton.dev/status-context": "demo-pipeline"
    "tekton.dev/status-description": "this is a test"
spec:
  pipelineRef:
    name: demo-pipeline
  serviceAccountName: 'default'
  resources:
  - name: source
    resourceSpec:
      type: git
      params:
        - name: revision
          value: insert revision
        - name: url
          value: https://github.com/this/repo
```

The revision here should be the full commit SHA from the HEAD of a branch associated with a Pull Request.

The annotations are:

<table style="width=100%" border="1">
  <tr>
    <th>Name</th>
    <th>Description</th>
    <th>Required</th>
    <th>Default</th>
  </tr>
  <tr>
    <th>
      tekton.dev/git-status
    </th>
    <td>
      This indicates that this <code>PipelineRun</code> should trigger commit-status notifications.
    </td>
    <td><b>Yes</b></td>
    <td></td>
  </tr>
  <tr>
    <th>
      tekton.dev/status-context
    </th>
    <td>
      This is the <a href="https://developer.github.com/v3/repos/statuses/#create-a-status">context</a> that will be reported, you can require named contexts in your branch protection rules.
    </td>
    <td>No</td>
    <td>"default"</td>
  </tr>
  <tr>
    <th>
      tekton.dev/status-description
    </th>
    <td>
      This is used as the description of the context, not the commit.
    </td>
    <td>No</td>
    <td>""</td>
  </tr>
  <tr>
    <th>
     tekton.dev/status-target-url
    </th>
    <td>
      If provided, then this will be linked in the GitHub web UI, this could be used to link to logs or output.
    </td>
    <td>No</td>
    <td>""</td>
  </tr>
  <tr>
    <th>
     tekton.dev/git-repo
    </th>
    <td>
      If provided together with <i>tekton.dev/git-revision</i> detecting the git repository from PiplineResource is skipped and given repository url is used
    </td>
    <td>No</td>
    <td></td>
  </tr>
  <tr>
    <th>
     tekton.dev/git-revision
    </th>
    <td>
      If provided together with <i>tekton.dev/git-repo</i> detecting the git repository from PiplineResource is skipped and given commit sha is used
    </td>
    <td>No</td>
    <td></td>
  </tr>
</table>

## Detecting the Git Repository

Currently, this uses a simple mechanism to find the Git repository and SHA to update the status of.

It looks for a single `PipelineResource` of type `git` and pulls the *url* and *revision* from there.

If no suitable `PipelineResource` is found, then this will be logged as an
error, and _not_ retried.

To override this behaviour (e.g. to support `Pipelines` which uses the _git-clone_ `Task`)
provide a git repository url and a commit sha via annotations
## Private Git repository hosts

You'll need to configure the deployment:

```yaml
env:
  - name: WATCH_NAMESPACE
    valueFrom:
      fieldRef:
        fieldPath: metadata.namespace
  - name: POD_NAME
    valueFrom:
      fieldRef:
        fieldPath: metadata.name
  - name: OPERATOR_NAME
    value: "commit-status-tracker"
  - name: GIT_DRIVERS
    value: "gl.example.com=gitlab"
```

If you are running with an untrusted SSL certificate, then you'll need to
slightly tweak the command:

```
containers:
  - name: commit-status-tracker
    command:
    - commit-status-tracker
    - --insecure
```

This `--insecure` is the same as curl's `-k/--insecure` in that it disables TLS
certificate verification, do not use this if you don't need to.

# Customizing the secret used to authenticate requests

It's possible to provide an environment variable `STATUS_TRACKER_SECRET` to
override the default secret name which is `commit-status-tracker-git-secret`.

## Prerequisites

- [go][go_tool] version v1.16+.
- [docker][docker_tool] version 17.03+
- [kubectl][kubectl_tool] v1.11.3+
- [operator-sdk][operator_sdk] v1.15
- Access to a Kubernetes v1.11.3+ cluster

## Getting Started

### Cloning the repository

Checkout the Operator repository

```
$ git clone https://github.com/tektoncd/experimental.git
$ cd experimental/commit-status-tracker
```
### Pulling the dependencies

Run the following command

```
$ go mod tidy
```

### Building the operator

Build the operator image and push it to a public registry, such as quay.io:

```
$ make IMAGE=quay.io/example-inc/commit-status-tracker:v0.0.1 docker-push
```

### Using the image

```shell
$ make IMAGE=quay.io/example-inc/commit-status-tracker:v0.0.1 bundle
```

**NOTE** The `quay.io/example-inc/commit-status-tracker:v0.0.1` is an example. You should build and push the image for your repository.

### Installing

You *must* have Tekton [Pipeline](https://github.com/tektoncd/pipeline/) installed before installing this operator:

```shell
$ kubectl apply -f kubectl apply -f https://github.com/tektoncd/pipeline/releases/download/v0.13.2/release.yaml
```

And then you can install the statuses operator with:

```shell
$ make IMAGE=quay.io/example-inc/commit-status-tracker:v0.0.1 deploy
```

### Uninstalling

```shell
$ make IMAGE=quay.io/example-inc/commit-status-tracker:v0.0.1 undeploy
```

### Troubleshooting

Use the following command to check the operator logs.

```shell
$ kubectl logs deployments/commit-status-tracker-controller-manager -n commit-status-tracker-system -c manager
```

### Running Tests

```shell
$ make test
```

[go_tool]: https://golang.org/dl/
[kubectl_tool]: https://kubernetes.io/docs/tasks/tools/install-kubectl/
[docker_tool]: https://docs.docker.com/install/
[operator_sdk]: https://github.com/operator-framework/operator-sdk
