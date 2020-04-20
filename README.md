# CREATING YOUR FIRST KUBERNETES OPERATOR


## Prerequisites
---

1. Install Golang

```
https://golang.org/doc/install
```

2. Setup go workspace

```
mkdir -p $HOME/go
go env -w GOPATH=$HOME/go
```

3. Install the Operator SDK CLI

```
$ brew install operator-sdk
```


## Create a new project
---

1. Use the CLI to create a new memcached-operator project

```
mkdir -p $GOPATH/src/github.com/demo/
cd $GOPATH/src/github.com/demo/
export GO111MODULE=on
operator-sdk new memcached-operator
```

2. Open the project with visual studio code

```
cd memcached-operator
code .
```

3. Install dependencies by run

```
go mod tidy
```

4. The following table describes a basic rundown of each generated file/directory.


| File/Folders   | Purpose                           |
| :---           | :--- |
| cmd       | Contains `manager/main.go` which is the main program of the operator. This instantiates a new manager which registers all custom resource definitions under `pkg/apis/...` and starts all controllers under `pkg/controllers/...`  . |
| pkg/apis | Contains the directory tree that defines the APIs of the Custom Resource Definitions(CRD). Users are expected to edit the `pkg/apis/<group>/<version>/<kind>_types.go` files to define the API for each resource type and import these packages in their controllers to watch for these resource types.|
| pkg/controller | This pkg contains the controller implementations. Users are expected to edit the `pkg/controller/<kind>/<kind>_controller.go` to define the controller's reconcile logic for handling a resource type of the specified `kind`. |
| build | Contains the `Dockerfile` and build scripts used to build the operator. |
| deploy | Contains various YAML manifests for registering CRDs, setting up [RBAC][RBAC], and deploying the operator as a Deployment.
| go.mod go.sum | The [Go mod][go_mod] manifests that describe the external dependencies of this operator. |
| vendor | The golang [vendor][Vendor] directory that contains local copies of external dependencies that satisfy Go imports in this project. [Go modules][go_mod] manages the vendor directory directly. This directory will not exist unless the project is initialized with the `--vendor` flag, or `go mod vendor` is run in the project root. |

[RBAC]: https://kubernetes.io/docs/reference/access-authn-authz/rbac/
[Vendor]: https://golang.org/cmd/go/#hdr-Vendor_Directories
[go_mod]: https://github.com/golang/go/wiki/Modules


## Add a new Custom Resource Definition
---

1. Add a new Custom Resource Definition (CRD) API called Memcached, with APIVersion cache.example.com/v1alpha1 and Kind Memcached

```
operator-sdk add api --api-version=cache.example.com/v1alpha1 --kind=Memcached
```

2. This will scaffold the Memcached resource API under pkg/apis/cache/v1alpha1/...


## Define the Memcached spec and status
---

1. You can find the generated CRD by run

```
cat deploy/crds/cache.example.com_memcacheds_crd.yaml
```

2. Let's Modify the spec and status of the Memcached Custom Resource (pkg/apis/cache/v1alpha1/memcached_types.go)

```
type MemcachedSpec struct {
    // Size is the size of the memcached deployment
    Size int32 `json:"size"`
}
type MemcachedStatus struct {
    // Nodes are the names of the memcached pods 
    Nodes []string `json:"nodes"`
}
```

3. After modifying the *_types.go file always run the following command to update the generated code for that resource type

```
operator-sdk generate k8s
```

4. Also run the following command in order to automatically generate the CRDs

```
operator-sdk generate crds
```

5. You can see the changes applied in deploy/crds/cache.example.com_memcacheds_crd.yaml

```
spec:
    description: MemcachedSpec defines the desired state of Memcached
    properties:
    size:
        description: Size is the size of the memcached deployment
        format: int32
        type: integer
    required:
    - size
    type: object
status:
    description: MemcachedStatus defines the observed state of Memcached
    properties:
    nodes:
        description: Nodes are the names of the memcached pods
        items:
        type: string
        type: array
```


## Add a new Controller
---

1. Add a new Controller to the project that will watch and reconcile the Memcached resource by run

```
operator-sdk add controller --api-version=cache.example.com/v1alpha1 --kind=Memcached
```

2. This will scaffold a new Controller implementation under pkg/controller/memcached/...


## Configure Controller Logic
---

