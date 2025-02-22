# Che/CodeReady Workspaces Operator

[![codecov](https://codecov.io/gh/eclipse-che/che-operator/branch/main/graph/badge.svg?token=IlYvrVU5nB)](https://codecov.io/gh/eclipse-che/che-operator)

Che/CodeReady workspaces operator uses [Operator SDK](https://github.com/operator-framework/operator-sdk) and [Go Kube client](https://github.com/kubernetes/client-go) to deploy, update and manage K8S/OpenShift resources that constitute a single or multi-user Eclipse Che/CodeReady Workspaces cluster.

The operator watches for a Custom Resource of Kind `CheCluster`, and operator controller executes its business logic when a new Che object is created, namely:

* creates k8s/OpenShift objects
* verifies successful deployment of Postgres, Devfile/Plugin registries and Che server
* updates CR spec and status (passwords, URLs, provisioning statuses etc.)
* continuously watches CR, update Che ConfigMap accordingly and schedule a new Che deployment
* changes state of certain objects depending on CR fields:
    * turn on/off TLS mode (reconfigure routes, update ConfigMap)
    * turn on/off OpenShift oAuth (login with OpenShift in Che) (create identity provider, oAuth client, update Che ConfigMap)
* etc

Che operator is implemented using [operator framework](https://github.com/operator-framework) and the Go programming language. Eclipse Che configuration defined using a custom resource definition object and stored in the custom Kubernetes resource named `checluster`(or plural `checlusters`) in the Kubernetes API group `org.eclipse.che`. Che operator extends Kubernetes API to embed Eclipse Che to Kubernetes cluster in a native way.

## CheCluster custom resource

Che operator deploys Eclipse Che using configuration stored in the Kubernetes custom resource(CR). CR object structure defined in the code using `api/v1/checluster_types.go` file. Field name defined using the serialization tag `json`, for example `json:"openShiftoAuth"`. Che operator default CR sample is stored in the `config/samples/org.eclipse.che_v1_checluster.yaml`. This file should be directly modified if you want to apply new fields with default values, or in case of changing default values for existing fields.
Also, you can apply in the field comments Openshift UI annotations: to display some
interactive information about these fields on the Openshift UI.
For example:

```go
	// +operator-sdk:csv:customresourcedefinitions:type=status
	// +operator-sdk:csv:customresourcedefinitions:type=status,displayName="Eclipse Che URL"
	// +operator-sdk:csv:customresourcedefinitions:type=status,xDescriptors="urn:alm:descriptor:org.w3:link"
```

This comment-annotations displays clickable link on the Openshift ui with a text "Eclipse Che URL"

It is mandatory to update the OLM bundle after modification of the CR sample to deploy Eclipse Che using OLM.

## Build and push custom Che operator image

1. Export globally environment variables:

```bash
$ export IMAGE_REGISTRY_USER_NAME=<IMAGE_REGISTRY_USER_NAME> && \
  export IMAGE_REGISTRY_HOST=<IMAGE_REGISTRY_HOST>
```

Where:
- `IMAGE_REGISTRY_USER_NAME` - docker image registry account name.
- `IMAGE_REGISTRY_HOST` - docker image registry hostname, for example: "docker.io", "quay.io". Host could be with a non default port: localhost:5000, 127.0.0.1:3000 and etc.

2. Run VSCode task `Build and push custom che-operator image` or use the terminal:

```bash
$ make docker-build docker-push IMG="${IMAGE_REGISTRY_HOST}/${IMAGE_REGISTRY_USER_NAME}/che-operator:next"
```

## Deploy Che operator using make

che-operator MAKE file provides ability to install che-operator(VSCode task `Deploy che-operator`):

```bash
$ make deploy IMG=\"${IMAGE_REGISTRY_HOST}/${IMAGE_REGISTRY_USER_NAME}/che-operator:next\"

$ kubectl apply -f config/samples/org.eclipse.che_v1_checluster.yaml -n <NAMESPACE>
```

Undeploy che-operator(VSCode task `UnDeploy che-operator`):

```bash
$ make undeploy
```

### Deploy Che operator with chectl

To deploy Che operator you can use [chectl](https://github.com/che-incubator/chectl). It has got two installer types corresponding to Che operator: `operator` and `olm`. With the `--installer operator` chectl reuses copies of Che operator deployment and roles (cluster roles) YAMLs, CR, CRD from the `deploy` directory of the project. With `--installer olm` chectl uses catalog source index image with olm bundles from the `bundle` directory.

#### Deploy Che operator with chectl using `--installer operator` flag

1. Build your custom operator image, see [How to Build che-operator Image](#build-and-push-custom-che-operator-image).

2. Deploy Eclipse Che on a running k8s cluster:

```bash
$ chectl server:deploy --installer operator -p <PLATFORM> --che-operator-image=${IMAGE_REGISTRY_HOST}/${IMAGE_REGISTRY_USER_NAME}/che-operator:next
```

Where:
- `PLATFORM` - k8s platform supported by chectl.

If you have changed Che operator deployment, roles, cluster roles, CRD or CR then you must use `--templates` flag to point chectl to modified Che operator templates. Use make command to prepare chectl templates folder:

```bash
$ make chectl-templ TARGET=<SOME_PATH>/che-operator
```

Execute chectl:

```bash
$ chectl server:deploy --installer operator -p <PLATFORM> --che-operator-image=${IMAGE_REGISTRY_HOST}/${IMAGE_REGISTRY_USER_NAME}/che-operator:next --templates <SOME_PATH>/templates
```

#### Deploy Che operator with chectl using `--installer olm` flag

1. Build your custom operator image, see [How to Build Che operator Image](#build-and-push-custom-che-operator-image).

2. Update OLM files:

```bash
$ make update-resources -s
```

3. Build catalog source and bundle images:

```bash
$ olm/buildCatalog.sh -c <CHANNEL> -i <CATALOG_IMAGE>
```

4. Create a custom catalog source yaml (update strategy is workaround for https://github.com/operator-framework/operator-lifecycle-manager/issues/903):

```yaml
apiVersion:  operators.coreos.com/v1alpha1
kind:         CatalogSource
metadata:
  name:         eclipse-che-custom-catalog
  namespace:    eclipse-che
spec:
  image:         <CATALOG_IMAGE>
  sourceType:  grpc
  updateStrategy:
    registryPoll:
      interval: 5m
```

5. Deploy Che operator:


```bash
$ chectl server:deploy --installer=olm --platform=<PLATFORM> --catalog-source-yaml <PATH_TO_CUSTOM_CATALOG_SOURCE_YAML> --olm-channel=next --package-manifest-name=eclipse-che-preview-<openshift|kubernetes>
```

## Update Che operator deployment

### Edit checluster custom resource using a command-line interface (terminal)

You can modify any Kubernetes object using the UI (for example OpenShift web console) or you can also modify Kubernetes objects using the terminal:

```bash
$ kubectl edit checluster eclipse-che -n <ECLIPSE-CHE-NAMESPACE>
```

or:

```bash
$ kubectl patch checluster/eclipse-che --type=merge -p '<PATCH_JSON>' -n <ECLIPSE-CHE-NAMESPACE>
```

### Update checluster using chectl

You can update Che configuration using the `chectl server:update` command providing `--cr-patch` flag. See [chectl](https://github.com/che-incubator/chectl) for more details.

## Development

### Debug Che operator

You can run/debug this operator on your local machine (without deploying to a k8s cluster).

Go client grabs kubeconfig either from InClusterConfig or `~/.kube` locally. Make sure  your current kubectl context points to a target cluster and namespace and a current user can create objects in a target namespace.

```bash
`make debug ECLIPSE_CHE_NAMESPACE=<ECLIPSE-CHE-NAMESPACE> ECLIPSE_CHE_CR=<CUSTOM_RESOURCE_PATH>
```

Where:
* `ECLIPSE-CHE-NAMESPACE` - namespace name to deploy Che operator into, default is `che`
* `CUSTOM_RESOURCE` - path to custom resource yaml, default is `./config/samples/org.eclipse.che_v1_checluster.yaml`

Use VSCode debug configuration `Che Operator` to attach to the running process.

To uninstall che-operator use:

```bash
$ make uninstall
```

And then interrupt debug process by `CTRL+C`.

### Run unit tests

Che operator covered with unit tests. Some of them uses mocks. To run tests use VSCode task `Run che-operator tests` or run in the terminal in the root of the project:

```bash
$ go test -mod=vendor -v ./...
```

### Compile Che operator code

The operator will be compiled to the binary `/tmp/che-operator/che-operator`.
This command is useful to make sure that che-operator is still compiling after your changes. Run VSCode task: `Compile che-operator code` or use the terminal:

```bash
$ GOOS=linux GOARCH=${ARCH} CGO_ENABLED=0 GO111MODULE=on go build -mod=vendor -a -o /tmp/che-operator/che-operator main.go
```

### Format code

Run the VSCode task: `Format che-operator code` or use the terminal:

```bash
$ go fmt ./...
```

### Fix imports

Run the VSCode task: `Fix che-operator imports` or use the terminal:

```bash
$ find . -not -path \"./vendor/*\" -name '*.go' -exec goimports -l -w {} \\;
```

### Update golang dependencies

Che operator uses Go modules and a vendor folder. Run the VSCode task: `Update che-operator dependencies` or use the terminal:

```bash
$ go mod tidy
$ go mod vendor
```

New golang dependencies in the vendor folder should be committed and included in the pull request.

Notice: freeze all new transitive dependencies using "replaces" go.mod file section to prevent CQ issues.

### Updating Custom Resource Definition file

Che cluster custom resource definition (CRD) defines Eclipse CheCluster custom resource object. It contains information about object structure, field types, field descriptions. CRD file is a YAML definition located in the folder `config/crd/bases`. These files are auto-generated, so do not edit it directly to update them. If you want to add new fields or fix descriptions in the CRDs, make your changes in the file `api/v1/checluster_types.go` and run VSCode task `Update CR/CRDs` or use the terminal:

```
$ make generate; make manifests
```

This command will update CRD files:
  - `config/crd/bases/org_v1_che_crd.yaml`

CRD beta yamls should be used for back compatibility with Openshift 3.

### Update OLM bundle

Sometimes, during development, you need to modify some YAML definitions in the `config` folder or Che cluster custom resource. There are most frequently changes which should be included to the new OLM bundle:
  - operator deployment `config/manager/manager.yaml`
  - operator roles/cluster roles permissions. They are defined like role/rolebinding or cluster role/rolebinding yamls in the `config` folder.
  - operator custom resource CR `config/crd/bases/org_v1_che_cr.yaml`. This file contains the default CheCluster sample. Also this file is the default OLM CheCluster sample.
  - Che cluster custom resource definition `api/v1/checluster_types.go`. For example you want to fix some properties description or apply new Che type properties with default values. These changes affect CRD `config/crd/bases/org_v1_che_crd.yaml`.
  - add Openshift ui annotations for Che types properties (`api/v1/checluster_types.go`) to display information or interactive elements on the Openshift user interface.

For all these cases it's a necessary to generate a new OLM bundle to make these changes working with OLM. Run the VSCode tasks `Update resources` or use the terminal:

```bash
$ make update-resources -s
```

Every changes will be included to the `bundle` folder and will override all previous changes. OLM bundle changes should be committed to the pull request.

To update a bundle without version incrementation and time update you can use env variables `NO_DATE_UPDATE` and `NO_INCREMENT`. For example, during development you need to update bundle a lot of times with changed che-operator deployment or role, rolebinding and etc, but you don't want to increment the bundle version and time creation, when all desired changes were completed:

```bash
$ make update-resources NO_DATE_UPDATE="true" NO_INCREMENT="true" -s
```

### Che operator PR checks

Documentation about all Che operator test cases can be found [here](https://github.com/eclipse-che/che-operator/tree/main/.ci/README.md)

### Generate go mocks.

Install mockgen tool:

```bash
$ GO111MODULE=on go get github.com/golang/mock/mockgen@v1.4.4
```

Generate new mock for go interface. Example:

```bash
$ mockgen -source=pkg/util/process.go -destination=mocks/pkg/util/process_mock.go -package mock_util

$ mockgen -source=pkg/controller/che/oauth_initial_htpasswd_provider.go \
          -destination=mocks/pkg/controller/che/oauth_initial_htpasswd_provider_mock.go \
          -package mock_che
```

See more: https://github.com/golang/mock/blob/master/README.md

### Validation licenses for runtime dependencies

che-operator is an Eclipse Foundation project. So we have to use only open source runtime dependencies with Eclipse compatible license https://www.eclipse.org/legal/licenses.php.
Runtime dependencies license validation process described here: https://www.eclipse.org/projects/handbook/#ip-third-party
To merge code with third party dependencies you have to follow process: https://www.eclipse.org/projects/handbook/#ip-prereq-diligence
When you are using new golang dependencies you have to validate the license for transitive dependencies too.
You can skip license validation for test dependencies.
All new dependencies you can find using git diff in the go.sum file.

Sometimes in the go.sum file you can find few versions for the same dependency:

```go.sum
...
github.com/go-openapi/analysis v0.18.0/go.mod h1:IowGgpVeD0vNm45So8nr+IcQ3pxVtpRoBWb8PVZO0ik=
github.com/go-openapi/analysis v0.19.2/go.mod h1:3P1osvZa9jKjb8ed2TPng3f0i/UY9snX6gxi44djMjk=
...
```

In this case will be used only one version(the highest) in the runtime, so you need to validate license for only one of them(the latest).
But also you can find module path https://golang.org/ref/mod#module-path with major version suffix in the go.sum file:

```go.sum
...
github.com/evanphx/json-patch v4.11.0+incompatible/go.mod h1:50XU6AFN0ol/bzJsmQLiYLvXMP4fmwYFNcr97nuDLSk=
github.com/evanphx/json-patch/v5 v5.1.0/go.mod h1:G79N1coSVB93tBe7j6PhzjmR3/2VvlbKOFpnXhI9Bw4=
...
```

In this case we have the same dependency, but with different major versions suffix.
Main project module uses both these versions in runtime. So both of them should be validated.

Also there is some useful golang commands to take a look full list dependencies:

```bash
$ go list -mod=mod -m all
```

This command returns all test and runtime dependencies. Like mentioned above, you can skip test dependencies.

If you want to know dependencies relation you can build dependencies graph:

```bash
$ go mod graph
```

> IMPORTANT: Dependencies validation information should be stored in the `DEPENDENCIES.md` file.
