# Develop a New Chaos

After [preparing the development environment](./setup_env.md), let's develop a new type of chaos, HelloWorldChaos, which only prints a "hello world" message to log. Generally, to add a new chaos type for Chaos Mesh, you need the following steps:

1. [Add the chaos object in controller](#add-the-chaos-object-in-controller)
2. [Register the CRD](#register-the-crd)
3. [Implement the schema type](implement-the-schema-type)
4. [Make the docker image](#make-the-docker-image)
5. [Run-chaos](#run-chaos)

## Add the chaos object in controller

In Chaos Mesh, all chaos types are managed by the controller manager. To add a new chaos type, you need to start from adding the corresponding reconciler type in the controller, as instructed in the following steps:

1. Add the HelloWorldChaos object in the controler manager [main.go](https://github.com/pingcap/chaos-mesh/blob/master/cmd/controller-manager/main.go#L104),

**Note:** You will notice existing chaos types such as PodChaos, NetworkChaos and IoChaos. Add the new type below them.

```golang
	if err = (&controllers.HelloWorldChaosReconciler{
		Client: mgr.GetClient(),
		Log:    ctrl.Log.WithName("controllers").WithName("HelloWorldChaos"),
	}).SetupWithManager(mgr); err != nil {
		setupLog.Error(err, "unable to create controller", "controller", "HelloWorldChaos")
		os.Exit(1)
	}
```

1. Under [controllers](https://github.com/pingcap/chaos-mesh/tree/master/controllers), create a `helloworldchaos_controller.go` file and edit it as below:


```golang
package controllers

import (
	"github.com/go-logr/logr"

	chaosmeshv1alpha1 "github.com/pingcap/chaos-mesh/api/v1alpha1"

	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
)

// HelloWorldChaosReconciler reconciles a HelloWorldChaos object
type HelloWorldChaosReconciler struct {
	client.Client
	Log logr.Logger
}

// +kubebuilder:rbac:groups=pingcap.com,resources=helloworldchaos,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=pingcap.com,resources=helloworldchaos/status,verbs=get;update;patch

func (r *HelloWorldChaosReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {
	logger := r.Log.WithValues("reconciler", "helloworldchaos")

        //  the main logic of `HelloWorldChaos`, it prints a log `Hello World!` and returns nothing.
	logger.Info("Hello World!")

	return ctrl.Result{}, nil
}

func (r *HelloWorldChaosReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
	//exports `HelloWorldChaos` object, which represents the yaml schema content the user applies.
		For(&chaosmeshv1alpha1.HelloWorldChaos{}).
		Complete(r)
}
```

> **Note:**
>
> The comment `// +kubebuilder:rbac:groups=pingcap.com...` is an authority control mechanism that decides which account can access this reconciler. To make it be accessible by dashboard and chaos-controller-manager, we need to modify  [collector-rbac.yaml](https://github.com/pingcap/chaos-mesh/blob/master/helm/chaos-mesh/templates/collector-rbac.yaml) and [controller-manager-rbac.yaml](https://github.com/pingcap/chaos-mesh/blob/master/helm/chaos-mesh/templates/controller-manager-rbac.yaml) accordingly.

## Register the CRD

The HelloWorldChaos object is a customer resource object in k8s. This means you need to register the corresponding CRD in the K8s API. To do this, modify [kustomization.yaml](https://github.com/pingcap/chaos-mesh/blob/master/config/crd/kustomization.yaml) by adding the corresponding line as shown below:

```yaml
resources:
- bases/pingcap.com_podchaos.yaml
- bases/pingcap.com_networkchaos.yaml
- bases/pingcap.com_iochaos.yaml
- bases/pingcap.com_helloworldchaos.yaml  # this is the new line
```

## Implement the schema type

To implement the schema type for the new chaos object, add `helloworldchaos_types.go` in the [api directory](https://github.com/pingcap/chaos-mesh/tree/master/api/v1alpha1) and modify it as below:


```golang
package v1alpha1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// +kubebuilder:object:root=true

// HelloWorldChaos is the Schema for the helloworldchaos API
type HelloWorldChaos struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`
}

// +kubebuilder:object:root=true

// HelloWorldChaosList contains a list of HelloWorldChaos
type HelloWorldChaosList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []HelloWorldChaos `json:"items"`
}

func init() {
	SchemeBuilder.Register(&HelloWorldChaos{}, &HelloWorldChaosList{})
}
```

With this file added, the `HelloWorldChaos` schema type is defined and can be called by the following yaml lines:

```yaml
apiVersion: pingcap.com/v1alpha1
kind: HelloWorldChaos
metadata:
  name: <name-of-this-resource>
  namespace: <ns-of-this-resource>
```

## Make the image

Having the object successfully added, we can make docker image and push it to your registry:

```
make
make docker-push
```

Please note that the default `DOCKER_REGISTRY` is `localhost:5000`, which is preset in `hack/kind-cluster-build.sh`. You can overwrite it to any registry to which you have access permission.

## Run Chaos

You are almost there. In this step, we will pull the image and apply it for testing.

**Note:**

Before you pull any image for chaos-mesh (using `helm install` or `helm upgrade`), modify [values.yaml](https://github.com/pingcap/chaos-mesh/blob/master/helm/chaos-mesh/values.yaml) of helm template to replace the default image with what you just pushed to your local registry. In the case, the template uses `pingcap/chaos-mesh:latest` as the default target registry, so we need to replace it with `localhost:5000`, as shown below:

```yaml
clusterScoped: true

# Also see clusterScoped and controllerManager.serviceAccount
rbac:
  create: true

controllerManager:
  serviceAccount: chaos-controller-manager
  ...
  image: localhost:5000/pingcap/chaos-mesh:latest
  ...
chaosDaemon:
  image: localhost:5000/pingcap/chaos-daemon:latest
  ...
dashboard:
  image: localhost:5000/pingcap/chaos-dashboard:latest
  ...
```

Now take the following steps to run chaos:

1. Get the related custom resource type for chaos-mesh:

```
kubectl apply -f manifests/
kubectl get crd podchaos.pingcap.com
```

3. Install Chaos Mesh:

```
helm install helm/chaos-mesh --name=chaos-mesh --namespace=chaos-testing --set chaosDaemon.runtime=containerd --set chaosDaemon.socketPath=/run/containerd/containerd.sock
kubectl get pods --namespace chaos-testing -l app.kubernetes.io/instance=chaos-mesh
```
The arguments `--set chaosDaemon.runtime=containerd --set chaosDaemon.socketPath=/run/containerd/containerd.sock` is used to to support network chaos on kind.

4. Create chaos.yaml with the lines below:

```yaml
apiVersion: pingcap.com/v1alpha1
kind: HelloWorldChaos
metadata:
  name: hello-world
  namespace: chaos-testing
```

5. Apply the chaos.

```
kubectl apply -f chaos.yaml
kubectl get HelloWorldChaos -n chaos-testing
```

Now you should be able to check the `Hello World!` result in the log:

```
kubectl logs chaos-controller-manager-{pod-post-fix} -n chaos-testing
# {pod-post-fix} is a random string generated by k8s
```

## Next steps

Congratulations! You have just added a chaos type for Chaos Mesh successfully. Let us know if you run into any issues during the process. If you feel like doing other type of contributions , refer to [Add facilities to chaos daemon].