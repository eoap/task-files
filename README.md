# Task files

This repository contains task files to be used as remote task files.

It relies on `project.toml` file, examples can be found in:

* [https://github.com/eoap/advanced-tooling/blob/main/project.toml](https://github.com/eoap/advanced-tooling/blob/main/project.toml) 
* [https://github.com/eoap/dask-app-package/blob/main/project.toml](https://github.com/eoap/dask-app-package/blob/main/project.toml)

## Explanation of project.toml and its Usage in Taskfile

The `project.toml` defines configurations for **building, testing, and executing CWL tools and workflows**, supporting both **local and in-cluster** execution. 

The **Taskfile** automates build and test execution based on the `project.toml` configuration.

###  Understanding project.toml Structure

This TOML file is divided into sections:

* Project Metadata (`[project]`)
* Build Configuration (`[build]`)
* Tool Definitions (`[tools]`)
* Workflow Definitions (`[[workflows]]`)

Each tool and workflow includes test cases with parameters and execution settings.

#### [project] - Project Metadata

```toml
[project]
name = "my-cwl-project"
version = "1.0.0"
```

Defines the project’s name and version for identification.

#### [build] - Build Configuration

```toml
[build]
engine = "cluster"
```

Determines where the build runs:

* `"local"` → Uses Docker/Podman
* `"cluster"` → Uses Skaffold + Kaniko in Kubernetes

#### [build.local] - Local Build Settings

```toml
[build.local]
runtime = "docker"
registry = "ghcr.io/eoap"
```

* Specifies container runtime (`docker` or `podman`).
* Uses registry for storing images when pushing.

#### [build.cluster] - In-Cluster Build Settings

```toml
[build.cluster]
namespace = "eoap-advanced-tooling"
serviceAccount = "kaniko-sa"
registry = "ghcr.io/eoap"
secret = "kaniko-secret"
```

* Defines the Kubernetes namespace and service account for kaniko builds.
* Uses a registry and secret for authentication.

#### CWL CommandLineTools: [tools] Section

Each CWL tool has:

* A directory containing the tool code (`context`)
* A CWL workflow file with the tool's entry point (`path`)
* Test cases (tests)

Example: Crop Tool

```toml
[tools.crop]
context = "command-line-tools/crop"
path = "cwl-workflow/app-water-bodies-cloud-native.cwl#crop"
```

* Defines the CWL tool
* Specifies its directory (`context`) and CWL workflow path (`path`)


#### Tool Test Cases: [[tools.<tool>.tests]]

Each tool has one or more test cases, specifying:

* Test input parameters
* Execution settings
* Storage paths

Example: Crop Tool (Green Band Test)

```toml
[[tools.crop.tests]]
name = "crop-test-green"
description = "Test case 1 for crop tool - green band."
```

Defines a test case for the tool.

#### [tools.<tool>.tests.params] - Test Parameters

```toml
[tools.crop.tests.params]
item = "https://earth-search.aws.element84.com/.../S2B_10TFK_20210713_0_L2A"
aoi = "-121.399,39.834,-120.74,40.472"
epsg = "EPSG:4326"
band = "green"
```

Defines input parameters for the test.

#### [tools.<tool>.tests.execution.cluster] - In-Cluster Execution Settings

```toml
[tools.crop.tests.execution.cluster]
max_ram = "1G"
max_cores = 1
pod_serviceaccount = "calrissian-sa"
usage_report = "usage.json"
tool_logs_basepath = "logs"
```

* Specifies CPU & memory limits for Kubernetes.
* Uses a Kubernetes service account (`calrissian-sa`) for execution.

#### [tools.<tool>.tests.execution.paths] - File Storage Paths

```toml
[tools.crop.tests.execution.paths]
stdout = "results.json"
stderr = "app.log"
tmp_outdir_prefix = "tmp"
outdir = "results"
volume = "/calrissian/crop-green"
```

Defines where outputs and logs are stored.


#### Workflows: [[workflows]] Section

Workflows combine multiple tools into a complete processing pipeline.

Example: Water Bodies Detection Workflow

```toml
[[workflows]]
path = "cwl-workflow/app-water-bodies-cloud-native.cwl#water-bodies"
```

Defines the CWL workflow and entry point.

#### Workflow Test Cases: [[workflows.tests]]

Each workflow has test cases similar to tools.

```toml
[[workflows.tests]]
name = "water-detection-test-1"
description = "Test case 1 for water bodies detection."
```

Defines a test case for the workflow.

#### [workflows.tests.params] - Test Inputs

```toml
[workflows.tests.params]
stac_items = ["https://earth-search.aws.element84.com/..."]
aoi = "-121.399,39.834,-120.74,40.472"
epsg = "EPSG:4326"
```

Defines input STAC items for processing.

### How project.toml is Used in Taskfile.yaml

The Taskfile automates:

* Building images
* Updating CWL workflows
* Running tests locally or in Kubernetes

#### Building Images: build Task

* Cluster Build (`Skaffold` + `Kaniko`)

```yaml
tasks:
  build-cluster:
    silent: true
    cmds:
    - task: prepare-kaniko
    - task: build-kaniko
```

* Runs Skaffold & Kaniko only if `build.engine = cluster`.
* Generates `skaffold-auto.yaml` dynamically from `project.toml`.

#### Local Build (Docker/Podman)

```yaml
tasks:
  build-local:
    silent: true
    cmds:
      - |
        engine=$(tomlq -r '.build.engine' project.toml)
        if [ "$engine" != "local" ]; then exit 0; fi

        runtime=$(tomlq -r '.build.local.runtime' project.toml)
        registry=$(tomlq -r '.build.local.registry' project.toml)

        for tool in $(tomlq -r '.tools | keys[]' project.toml); do
            path=$(tomlq -r ".tools.$tool.context" project.toml)
            eval $runtime build -t $registry/$tool $path
        done
```

Reads `project.toml` to build tools using Docker/Podman.

#### Running Tests: test Task

Runs Tests for Each Workflow

```yaml
tasks:
  test:
    silent: true
    cmds:
      - |
        engine=$(tomlq -r '.build.engine' project.toml)
        tomlq -c '.workflows[]' project.toml | while read -r workflow; do
          path=$(tomlq -r ".workflows[$i].path" project.toml)

          if [ "$engine" = "cluster" ]; then
            cmd="calrissian --stdout ... --stderr ... ${path} params.yaml"
          elif [ "$engine" = "local" ]; then
            cmd="cwltool --tmp-outdir-prefix ... ${path} params.yaml"
          fi

          eval $cmd
        done
```

* Uses Calrissian in Kubernetes or CWLTool locally.
* Runs each workflow test dynamically.

## Project Taskfile

The Taskfile imports a remote Taskfile from https://github.com/eoap/task-files/tree/main/build-test and simply combines remote tasks

```yaml
version: '3'

includes:
  remote: https://raw.githubusercontent.com/eoap/task-files/c750354651eed547cc310aa7a856921e77ebc058/build-test/Taskfile.yaml

tasks:
  build:
  - task: remote:build

  prepare:
  - task: remote:prepare

  test:
  - task: remote:test

  test-all:
    cmds: 
    - task: test-crop
    - task: test-norm-diff
    - task: test-otsu
    - task: test-stac

  test-crop:
    cmds:
    - task: remote:test-tool
      vars:
        VAR: "crop"

  test-norm-diff:
    cmds:
    - task: remote:test-tool
      vars:
        VAR: "norm_diff"

  test-otsu:
    cmds:
    - task: remote:test-tool
      vars:
        VAR: "otsu" 

  test-stac:
    cmds:
    - task: remote:test-tool
      vars:
        VAR: "stac"
```

## Cluster configuration

Create an image pull secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: kaniko-secret
data:
  .dockerconfigjson: >-
    eyJh..ViJ9fX0=
type: kubernetes.io/dockerconfigjson
```

Create a ServiceAccount and mount the image pull secret:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kaniko-sa
imagePullSecrets:
  - name: kaniko-secret
```

Create a Role with:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: kaniko-role
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create", "get", "list", "watch", "delete"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get","list"]
```

And the RoleBinding to associate the Role to the ServiceAccount:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kaniko-rolebinding
subjects:
  - kind: ServiceAccount
    name: kaniko-sa
roleRef:
  kind: Role
  name: kaniko-role
  apiGroup: rbac.authorization.k8s.io
```