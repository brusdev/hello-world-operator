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
