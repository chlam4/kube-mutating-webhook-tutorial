# Kubernetes Mutating Webhook for Sidecar Injection

[![GoDoc](https://godoc.org/github.com/chlam4/kube-mutating-webhook-tutorial?status.svg)](https://godoc.org/github.com/chlam4/kube-mutating-webhook-tutorial)

This shows how to build and deploy a [MutatingAdmissionWebhook](https://kubernetes.io/docs/admin/admission-controllers/#mutatingadmissionwebhook-beta-in-19) that injects a Kubernetes secret as a volume into pods with corresponding annotations.

## Prerequisites

- [git](https://git-scm.com/downloads)
- [go](https://golang.org/dl/) version v1.12+
- [docker](https://docs.docker.com/install/) version 17.03+
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) version v1.11.3+
- Access to a Kubernetes v1.11.3+ cluster with the `admissionregistration.k8s.io/v1beta1` API enabled. Verify that by the following command:

```
kubectl api-versions | grep admissionregistration.k8s.io
```
The result should be:
```
admissionregistration.k8s.io/v1
admissionregistration.k8s.io/v1beta1
```

> Note: In addition, the `MutatingAdmissionWebhook` and `ValidatingAdmissionWebhook` admission controllers should be added and listed in the correct order in the admission-control flag of kube-apiserver.

## Build

1. Build binary

```
# make build
```

2. Build docker image
   
```
# IMAGE_TAG=v0.2 make build-image
```

3. push docker image

```
# IMAGE_TAG=v0.2 make push-image
```

> Note: log into the docker registry before pushing the image.

## Deploy

1. Create namespace `k8s-secret-injector` in which the injector webhook is deployed:

```
kubectl create ns k8s-secret-injector
```

2. Create a signed cert/key pair and store it in a Kubernetes `secret` that will be consumed by injector deployment:

```
./deployment/webhook-create-signed-cert.sh \
    --service k8s-secret-injector-webhook-svc \
    --secret k8s-secret-injector-webhook-certs \
    --namespace k8s-secret-injector
```

3. Patch the `MutatingWebhookConfiguration` by set `caBundle` with correct value from Kubernetes cluster:

```
cat deployment/mutatingwebhook.yaml | \
    deployment/webhook-patch-ca-bundle.sh > \
    deployment/mutatingwebhook-ca-bundle.yaml
```

4. Deploy resources:

```
kubectl apply -f deployment/configmap.yaml
kubectl apply -f deployment/deployment.yaml
kubectl apply -f deployment/service.yaml
kubectl apply -f deployment/mutatingwebhook-ca-bundle.yaml
```

## Verify

1. The k8s secret inject webhook should be running
```
$ kubectl -n k8s-secret-injector get pods
NAME                                                     READY   STATUS    RESTARTS   AGE
k8s-secret-injector-webhook-deployment-dbfd96b7f-nfjqr   1/1     Running   0          2m24s
$ kubectl -n k8s-secret-injector get deployment
NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
k8s-secret-injector-webhook-deployment   1/1     1            1           51m```
```

2. Label the desired namespace with `k8s-secret-injector=enabled`.  In the example below, we use `turbonomic` as the namespace.
```
$ kubectl label namespace turbonomic k8s-secret-injector=enabled
$ kubectl get namespace -L k8s-secret-injector
NAME                  STATUS   AGE    K8S-SECRET-INJECTOR
default               Active   350d
k8s-secret-injector   Active   87m
kube-node-lease       Active   350d
kube-public           Active   350d
kube-system           Active   350d
turbonomic            Active   350d   enabled
```

3. Verify that a Kubernetes secret has been injected as a volume
TBD