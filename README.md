# Hello World Operator
Hello World Operator is a [Kubernetes Operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) of
[Hello World HTTP](https://github.com/brusdev/hello-world-http).
It is an in-depth walkthrough of building and running a Go-based operator using the
[Operator SDK](https://sdk.operatorframework.io/).

## Install prerequisites
Install Go https://go.dev/doc/install.

Install Operator SDK CLI https://sdk.operatorframework.io/docs/installation/.

Install Docker https://docs.docker.com/get-docker/

## Initialize hello-world-operator
Use the Operator SDK CLI to initialize the hello-world-operator project:
```
$ mkdir -p hello-world-operator
$ cd hello-world-operator
$ operator-sdk init --domain brus.dev --repo github.com/brusdev/hello-world-operator
```

To learn about the project directory structure, see
[Kubebuilder project layout](https://book.kubebuilder.io/cronjob-tutorial/basic-project.html) doc.

The main program for the operator [main.go](./main.go) initializes and runs the Manager.

## Create HelloWorld API and controller
Create the Custom Resource Definition (CRD) API v1alpha1/HelloWorld and the controller to deploy and manage
[Hello World HTTP](https://github.com/brusdev/hello-world-http):
```
$ operator-sdk create api --group examples --version v1alpha1 --kind HelloWorld --plugins="deploy-image/v1-alpha" --image=quay.io/brusdev/hello-world-http:latest
```
This will scaffold the following items:
- the resource API [helloworld_types.go](./api/v1alpha1/helloworld_types.go)
- the controller [helloworld_controller.go](./controllers/helloworld_controller.go)
- the controller test [helloworld_controller_test.go](./controllers/helloworld_controller_test.go)
Excute the controller test using the command:
```
$ make test
```

## Configure container image registry
The Operator SDK CLI init command creates a [Makefile](./Makefile) that define three environment variables for
container image URLs:
- IMG for the operator container image URL;
- IMAGE_TAG_BASE for the container image base URL;
- BUNDLE_IMG for the bundle container image URL;
- CATALOG_IMG for the catalog container image URL;

The values of those environment variables must be valid container image URLs. Use a valid container registry domain
to define a valid container image URL, i.e. `quay.io/brusdev/hello-world-operator`. Create a free
[Quay.io](https://quay.io/) account to get a container registry domain.

Update the `IMG` value to `$(IMAGE_TAG_BASE):$(VERSION)` and the `IMAGE_TAG_BASE` value using a valid container registry
domain if the current value is incorrect, i.e. `quay.io/brusdev/hello-world-operator`.

## Build operator container image
Build the operator container image using the `docker-build` target defined in [Makefile](./Makefile):
```
$ make docker-build
```

## Push operator container image
Create a new public repository that matches the operator container image URL, i.e. `quay.io/brusdev/hello-world-operator`, to create a new repository in the Quay.io container registry follow the
[creating a repository guide](https://docs.quay.io/guides/create-repo.html).

Login to the container registry, i.e. to login the Quay.io container registry:
```
$ docker login quay.io
```

Push the operator container image using the `docker-push` target defined in [Makefile](./Makefile):
```
$ make docker-push
```

## Deploy operator
Login to a Kubernetes cluster or start [minikube](https://kubernetes.io/docs/tasks/tools/#minikube) to run a local Kubernetes:
```
$ minikube start
```
Install [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) or use minikube kubectl:
```
$ alias kubectl="minikube kubectl --"
```
Run the following to deploy the operator. This will also install the RBAC manifests from config/rbac.
```
$ make deploy
```

## Access operator metrics
Create a serice account to access operator metrics:
```
$ kubectl create serviceaccount metrics-reader
$ kubectl create clusterrolebinding metrics-reader --clusterrole=hello-world-operator-metrics-reader --serviceaccount=$(kubectl get serviceaccount metrics-reader -o jsonpath='{.metadata.namespace}'):metrics-reader
```
Test the access to metrics using busybox:
```
$ kubectl run -it --rm busybox --image=busybox --restart=Never --overrides='{"spec":{"serviceAccount":"metrics-reader"}}'
If you don't see a command prompt, try pressing enter.
/ #
/ #
/ # wget -O - --header="Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" --no-check-certificate https://hello-world-operator-controller-manager-metrics-service.hello-world-operator-system:8443/metrics
```

## Create HelloWorld CR
Create a HelloWorld CR using the kubectl:
```
$ kubectl apply -f - <<EOF
apiVersion: examples.brus.dev/v1alpha1
kind: HelloWorld
metadata:
  name: helloworld-sample
spec:
  size: 3
EOF
```
Ensure that the helloworld operator creates the deployment for the HelloWorld CR with the correct size:
```
$ kubectl get deployment
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
helloworld-sample   3/3     3            3           1m
```
Check the pods and CR status to confirm the status is updated with the helloworld pod names:
```
$ kubectl get pods
NAME                                  READY     STATUS    RESTARTS   AGE
helloworld-sample-6fd7c98d8-7dqdr     1/1       Running   0          1m
helloworld-sample-6fd7c98d8-g5k7v     1/1       Running   0          1m
helloworld-sample-6fd7c98d8-m7vn7     1/1       Running   0          1m
```
Change the `spec.size` field in the HelloWorld CR from 3 to 1:
```
kubectl patch helloworld helloworld-sample -p '{"spec":{"size": 1}}' --type=merge
```
Confirm that the operator changes the deployment size:
```
$ kubectl get deployment
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
helloworld-sample   1/1     1            1           1m
```

## Expose helloworld pods
Expose helloworld pods with a [service](https://kubernetes.io/docs/concepts/services-networking/service/). Improve the
helloworld controller to create a service for avery helloworld deployment:
```
// Define a new service
svc := &corev1.Service{
    ObjectMeta: metav1.ObjectMeta{
        Name:      helloworld.Name,
        Namespace: helloworld.Namespace,
    },
    Spec: corev1.ServiceSpec{
        Selector: labelsForHelloWorld(helloworld.Name),
        Ports: []corev1.ServicePort{
            {
                Protocol:   "TCP",
                Port:       8080,
                TargetPort: intstr.FromInt(8080),
            },
        },
    },
}
```
Create a HelloWorld CR using the kubectl:
```
$ kubectl apply -f - <<EOF
apiVersion: examples.brus.dev/v1alpha1
kind: HelloWorld
metadata:
  name: helloworld-sample
spec:
  size: 1
EOF
```
Ensure that the helloworld operator creates the service for the HelloWorld deployment:
```
$ kubectl get service
NAME                TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
helloworld-sample   ClusterIP   10.98.63.22   <none>        8080/TCP   1m
```
Test the deployment and the service using busybox:
```
$ kubectl run -it --rm busybox --image=busybox --restart=Never --command -- wget -O - helloworld-sample:8080
Hello, World!
pod "busybox" deleted
```

## Set hello text
Add the `spec.text` field to set the HELLO_TEXT enviornment variable of the
[Hello World HTTP](https://github.com/brusdev/hello-world-http) container:
```
Env: []corev1.EnvVar{
    {
        Name:  "HELLO_TEXT",
        Value: helloworld.Spec.Text,
    },
},
```
Create a HelloWorld CR with a custom text using the kubectl:
```
$ kubectl apply -f - <<EOF
apiVersion: examples.brus.dev/v1alpha1
kind: HelloWorld
metadata:
  name: helloworld-sample
spec:
  size: 1
  text: Hello, brusdev!
EOF
```
Ensure that the helloworld operator creates the deployment with the `HELLO_TEXT` enviornment variable:
```
$ kubectl get deployment helloworld-sample -o jsonpath-as-json={.spec.template.spec.containers}
[
    [
        {
            "env": [
                {
                    "name": "HELLO_TEXT",
                    "value": "Hello, brusdev!"
                }
            ],
            "image": "quay.io/brusdev/hello-world-http:latest",
            "imagePullPolicy": "IfNotPresent",
            "name": "helloworld",
            "resources": {},
            "securityContext": {
                "allowPrivilegeEscalation": false,
                "capabilities": {
                    "drop": [
                        "ALL"
                    ]
                },
                "runAsNonRoot": true
            },
            "terminationMessagePath": "/dev/termination-log",
            "terminationMessagePolicy": "File"
        }
    ]
]
```
Test the deployment and the service with a custom text using busybox:
```
$ kubectl run -it --rm busybox --image=busybox --restart=Never --command -- wget -q -O - helloworld-sample:8080
Hello, brusdev!
pod "busybox" deleted
```
