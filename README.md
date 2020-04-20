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

The example Controller executes the following reconciliation logic for each Memcached CR:
 - Create a memcached Deployment if it doesn't exist
 - Ensure that the Deployment size is the same as specified by the Memcached CR spec
 - Update the Memcached CR status with the names of the memcached pods

1. Override the content of the pkg/controller/memcached/memcached_controller.go with the content below

```
package memcached

import (
	"context"
	"reflect"

	cachev1alpha1 "github.com/demo/memcached-operator/pkg/apis/cache/v1alpha1"

	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/types"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/controller"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
	"sigs.k8s.io/controller-runtime/pkg/handler"
	"sigs.k8s.io/controller-runtime/pkg/manager"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"
	logf "sigs.k8s.io/controller-runtime/pkg/runtime/log"
	"sigs.k8s.io/controller-runtime/pkg/source"
)

var log = logf.Log.WithName("controller_memcached")

// Add creates a new Memcached Controller and adds it to the Manager. The Manager will set fields on the Controller
// and Start it when the Manager is Started.
func Add(mgr manager.Manager) error {
	return add(mgr, newReconciler(mgr))
}

// newReconciler returns a new reconcile.Reconciler
func newReconciler(mgr manager.Manager) reconcile.Reconciler {
	return &ReconcileMemcached{client: mgr.GetClient(), scheme: mgr.GetScheme()}
}

// add adds a new Controller to mgr with r as the reconcile.Reconciler
func add(mgr manager.Manager, r reconcile.Reconciler) error {
	// Create a new controller
	c, err := controller.New("memcached-controller", mgr, controller.Options{Reconciler: r})
	if err != nil {
		return err
	}

	// Watch for changes to primary resource Memcached
	err = c.Watch(&source.Kind{Type: &cachev1alpha1.Memcached{}}, &handler.EnqueueRequestForObject{})
	if err != nil {
		return err
	}

	// TODO(user): Modify this to be the types you create that are owned by the primary resource
	// Watch for changes to secondary resource Pods and requeue the owner Memcached
	err = c.Watch(&source.Kind{Type: &appsv1.Deployment{}}, &handler.EnqueueRequestForOwner{
		IsController: true,
		OwnerType:    &cachev1alpha1.Memcached{},
	})
	if err != nil {
		return err
	}

	err = c.Watch(&source.Kind{Type: &corev1.Service{}}, &handler.EnqueueRequestForOwner{
		IsController: true,
		OwnerType:    &cachev1alpha1.Memcached{},
	})
	if err != nil {
		return err
	}

	return nil
}

var _ reconcile.Reconciler = &ReconcileMemcached{}

// ReconcileMemcached reconciles a Memcached object
type ReconcileMemcached struct {
	// TODO: Clarify the split client
	// This client, initialized using mgr.Client() above, is a split client
	// that reads objects from the cache and writes to the apiserver
	client client.Client
	scheme *runtime.Scheme
}

// Reconcile reads that state of the cluster for a Memcached object and makes changes based on the state read
// and what is in the Memcached.Spec
// TODO(user): Modify this Reconcile function to implement your Controller logic.  This example creates
// a Memcached Deployment for each Memcached CR
// Note:
// The Controller will requeue the Request to be processed again if the returned error is non-nil or
// Result.Requeue is true, otherwise upon completion it will remove the work from the queue.
func (r *ReconcileMemcached) Reconcile(request reconcile.Request) (reconcile.Result, error) {
	reqLogger := log.WithValues("Request.Namespace", request.Namespace, "Request.Name", request.Name)
	reqLogger.Info("Reconciling Memcached.")

	// Fetch the Memcached instance
	memcached := &cachev1alpha1.Memcached{}
	err := r.client.Get(context.TODO(), request.NamespacedName, memcached)
	if err != nil {
		if errors.IsNotFound(err) {
			// Request object not found, could have been deleted after reconcile request.
			// Owned objects are automatically garbage collected. For additional cleanup logic use finalizers.
			// Return and don't requeue
			reqLogger.Info("Memcached resource not found. Ignoring since object must be deleted.")
			return reconcile.Result{}, nil
		}
		// Error reading the object - requeue the request.
		reqLogger.Error(err, "Failed to get Memcached.")
		return reconcile.Result{}, err
	}

	// Check if the Deployment already exists, if not create a new one
	deployment := &appsv1.Deployment{}
	err = r.client.Get(context.TODO(), types.NamespacedName{Name: memcached.Name, Namespace: memcached.Namespace}, deployment)
	if err != nil && errors.IsNotFound(err) {
		// Define a new Deployment
		dep := r.deploymentForMemcached(memcached)
		reqLogger.Info("Creating a new Deployment.", "Deployment.Namespace", dep.Namespace, "Deployment.Name", dep.Name)
		err = r.client.Create(context.TODO(), dep)
		if err != nil {
			reqLogger.Error(err, "Failed to create new Deployment.", "Deployment.Namespace", dep.Namespace, "Deployment.Name", dep.Name)
			return reconcile.Result{}, err
		}
		// Deployment created successfully - return and requeue
		// NOTE: that the requeue is made with the purpose to provide the deployment object for the next step to ensure the deployment size is the same as the spec.
		// Also, you could GET the deployment object again instead of requeue if you wish. See more over it here: https://godoc.org/sigs.k8s.io/controller-runtime/pkg/reconcile#Reconciler
		return reconcile.Result{Requeue: true}, nil
	} else if err != nil {
		reqLogger.Error(err, "Failed to get Deployment.")
		return reconcile.Result{}, err
	}

	// Ensure the deployment size is the same as the spec
	size := memcached.Spec.Size
	if *deployment.Spec.Replicas != size {
		deployment.Spec.Replicas = &size
		err = r.client.Update(context.TODO(), deployment)
		if err != nil {
			reqLogger.Error(err, "Failed to update Deployment.", "Deployment.Namespace", deployment.Namespace, "Deployment.Name", deployment.Name)
			return reconcile.Result{}, err
		}
	}

	// Check if the Service already exists, if not create a new one
	// NOTE: The Service is used to expose the Deployment. However, the Service is not required at all for the memcached example to work. The purpose is to add more examples of what you can do in your operator project.
	service := &corev1.Service{}
	err = r.client.Get(context.TODO(), types.NamespacedName{Name: memcached.Name, Namespace: memcached.Namespace}, service)
	if err != nil && errors.IsNotFound(err) {
		// Define a new Service object
		ser := r.serviceForMemcached(memcached)
		reqLogger.Info("Creating a new Service.", "Service.Namespace", ser.Namespace, "Service.Name", ser.Name)
		err = r.client.Create(context.TODO(), ser)
		if err != nil {
			reqLogger.Error(err, "Failed to create new Service.", "Service.Namespace", ser.Namespace, "Service.Name", ser.Name)
			return reconcile.Result{}, err
		}
	} else if err != nil {
		reqLogger.Error(err, "Failed to get Service.")
		return reconcile.Result{}, err
	}

	// Update the Memcached status with the pod names
	// List the pods for this memcached's deployment
	podList := &corev1.PodList{}
	listOpts := []client.ListOption{
		client.InNamespace(memcached.Namespace),
		client.MatchingLabels(labelsForMemcached(memcached.Name)),
	}
	err = r.client.List(context.TODO(), podList, listOpts...)
	if err != nil {
		reqLogger.Error(err, "Failed to list pods.", "Memcached.Namespace", memcached.Namespace, "Memcached.Name", memcached.Name)
		return reconcile.Result{}, err
	}
	podNames := getPodNames(podList.Items)

	// Update status.Nodes if needed
	if !reflect.DeepEqual(podNames, memcached.Status.Nodes) {
		memcached.Status.Nodes = podNames
		err := r.client.Status().Update(context.TODO(), memcached)
		if err != nil {
			reqLogger.Error(err, "Failed to update Memcached status.")
			return reconcile.Result{}, err
		}
	}

	return reconcile.Result{}, nil
}

// deploymentForMemcached returns a memcached Deployment object
func (r *ReconcileMemcached) deploymentForMemcached(m *cachev1alpha1.Memcached) *appsv1.Deployment {
	ls := labelsForMemcached(m.Name)
	replicas := m.Spec.Size

	dep := &appsv1.Deployment{
		ObjectMeta: metav1.ObjectMeta{
			Name:      m.Name,
			Namespace: m.Namespace,
		},
		Spec: appsv1.DeploymentSpec{
			Replicas: &replicas,
			Selector: &metav1.LabelSelector{
				MatchLabels: ls,
			},
			Template: corev1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{
					Labels: ls,
				},
				Spec: corev1.PodSpec{
					Containers: []corev1.Container{{
						Image:   "memcached:1.4.36-alpine",
						Name:    "memcached",
						Command: []string{"memcached", "-m=64", "-o", "modern", "-v"},
						Ports: []corev1.ContainerPort{{
							ContainerPort: 11211,
							Name:          "memcached",
						}},
					}},
				},
			},
		},
	}
	// Set Memcached instance as the owner of the Deployment.
	controllerutil.SetControllerReference(m, dep, r.scheme)
	return dep
}

// serviceForMemcached function takes in a Memcached object and returns a Service for that object.
func (r *ReconcileMemcached) serviceForMemcached(m *cachev1alpha1.Memcached) *corev1.Service {
	ls := labelsForMemcached(m.Name)
	ser := &corev1.Service{
		ObjectMeta: metav1.ObjectMeta{
			Name:      m.Name,
			Namespace: m.Namespace,
		},
		Spec: corev1.ServiceSpec{
			Selector: ls,
			Ports: []corev1.ServicePort{
				{
					Port: 11211,
					Name: m.Name,
				},
			},
		},
	}
	// Set Memcached instance as the owner of the Service.
	controllerutil.SetControllerReference(m, ser, r.scheme)
	return ser
}

// labelsForMemcached returns the labels for selecting the resources
// belonging to the given memcached CR name.
func labelsForMemcached(name string) map[string]string {
	return map[string]string{"app": "memcached", "memcached_cr": name}
}

// getPodNames returns the pod names of the array of pods passed in
func getPodNames(pods []corev1.Pod) []string {
	var podNames []string
	for _, pod := range pods {
		podNames = append(podNames, pod.Name)
	}
	return podNames
}
```


## Understand the Controller
---

- The first watch is for the Memcached type as the primary resource. For each Add/Update/Delete event the reconcile loop will be sent a reconcile Request (a namespace/name key) for that Memcached object:

```
err := c.Watch(
   &source.Kind{Type: &cachev1alpha1.Memcached{}},
   &handler.EnqueueRequestForObject{},
 )
```

- The next watch is for Deployments but the event handler will map each event to a reconcile Request for the owner of the Deployment. Which in this case is the Memcached object for which the Deployment was created. This allows the controller to watch Deployments as a secondary resource.

```
err := c.Watch(
    &source.Kind{Type: &appsv1.Deployment{}},
    &handler.EnqueueRequestForOwner{
        IsController: true,
        OwnerType:    &cachev1alpha1.Memcached{}},
    )
```

- Every Controller has a Reconciler object with a Reconcile() method that implements the reconcile loop. The reconcile loop is passed the Request argument which is a Namespace/Name key used to lookup the primary resource object, Memcached, from the cache

```
func (r *ReconcileMemcached) Reconcile(request reconcile.Request) (reconcile.Result, error) {
    // Lookup the Memcached instance for this reconcile request
    memcached := &cachev1alpha1.Memcached{}
    err := r.client.Get(context.TODO(), request.NamespacedName, memcached)
    ...
} 
```

## Run the operator locally (dev)
---

1. Before running the operator, the CRD must be registered with the Kubernetes apiserver:

```
kubectl create -f deploy/crds/cache.example.com_memcacheds_crd.yaml
```

2. Run the operator locally with the default kubernetes config file present at $HOME/.kube/config

```
operator-sdk run --local --watch-namespace=default
```


## Create a Memcached CR
---

1. Check the existent resources in your environment (in a new terminal)

```
kubectl get all
```

2. Create the example Memcached CR that was generated at deploy/crds/cache.example.com_v1alpha1_memcached_cr.yaml

```
kubectl apply -f deploy/crds/cache.example.com_v1alpha1_memcached_cr.yaml
```

3. Ensure that the memcached-operator creates a deployment/pods for the CR:

```
kubectl get all
```


## Update the size
---

1. Uppdate the desired size from 3 to 5 in the created CR by run 

```
kubectl edit Memcached example-memcached
```

3. Ensure that the memcached-operator creates a deployment for the CR:

```
kubectl get pods
```

## Deploy the Operator

1. Build a docker image for the operator

```
operator-sdk build leonjalfon1/memcached-operator:v0.0.1
```

2. Set the image in the "./deploy/operator.yaml" file

```
# Linux
sed -i 's|REPLACE_IMAGE|leonjalfon1/memcached-operator:v0.0.1|g' deploy/operator.yaml

# Mac
sed -i "" 's|REPLACE_IMAGE|leonjalfon1/memcached-operator:v0.0.1|g' deploy/operator.yaml
```

3. Push the operator image

```
docker push leonjalfon1/memcached-operator:v0.0.1
```

4. Setup RBAC and deploy the memcached-operator:

```
kubectl create -f deploy/service_account.yaml
kubectl create -f deploy/role.yaml
kubectl create -f deploy/role_binding.yaml
kubectl create -f deploy/operator.yaml
```

5. Verify that the memcached-operator pod is up and running:

```
kubectl get pods
```

6. Verify that the operator is running successfully by checking its logs.

```
kubectl logs memcached-operator-5b8888b455-sdxwg
```