# Kubernetes Mutating Admission Webhook for sidecar injection

This shows how to build and deploy a [MutatingAdmissionWebhook](https://kubernetes.io/docs/admin/admission-controllers/#mutatingadmissionwebhook-beta-in-19) that injects a specially built secret sidecar container into pod.

## Prerequisites

Kubernetes 1.9.0 or above with the `admissionregistration.k8s.io/v1beta1` API enabled. Verify that by the following command:
```
kubectl api-versions | grep admissionregistration.k8s.io/v1beta1
```
The result should be:
```
admissionregistration.k8s.io/v1beta1
```

In addition, the `MutatingAdmissionWebhook` and `ValidatingAdmissionWebhook` admission controllers should be added and listed in the correct order in the admission-control flag of kube-apiserver.

## Build

1. Setup dep

   The repo uses [dep](https://github.com/golang/dep) as the dependency management tool for its Go codebase. Install `dep` by the following command:
```
go get -u github.com/golang/dep/cmd/dep
```

2. Build and push docker image
   
```
./build
```

## Deploy

1. Create a signed cert/key pair and store it in a Kubernetes `secret` that will be consumed by sidecar deployment
```
./deployment/webhook-create-signed-cert.sh \
    --service sidecar-injector-webhook-svc \
    --secret sidecar-injector-webhook-certs \
    --namespace default
```

2. Patch the `MutatingWebhookConfiguration` by set `caBundle` with correct value from Kubernetes cluster
```
cat deployment/mutatingwebhook.yaml | \
    deployment/webhook-patch-ca-bundle.sh > \
    deployment/mutatingwebhook-ca-bundle.yaml
```

3. Deploy resources
```
kubectl create -f deployment/configmap.yaml
kubectl create -f deployment/deployment.yaml
kubectl create -f deployment/service.yaml
kubectl create -f deployment/mutatingwebhook-ca-bundle.yaml
```

## Verify

1. The sidecar inject webhook should be running
```
[root@mstnode ~]# kubectl get pods
NAME                                                  READY     STATUS    RESTARTS   AGE
sidecar-injector-webhook-deployment-bbb689d69-882dd   1/1       Running   0          5m
[root@mstnode ~]# kubectl get deployment
NAME                                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
sidecar-injector-webhook-deployment   1         1         1            1           5m
```

2. Label the desired namespace with `sidecar-injector=enabled`.  In the example below, we use `turbonomic` as the namespace.
```
kubectl label namespace turbonomic sidecar-injector=enabled
[root@mstnode ~]# kubectl get namespace -L sidecar-injector
NAME          STATUS    AGE       SIDECAR-INJECTOR
kube-public   Active    18h
kube-system   Active    18h
turbonomic    Active    18h       enabled
```

3. To take advantage of the sidecar for pulling secrets, a specially built probe and operator are required.
These changes will be later incorporated into the official images.

3a. Use the following image for operator.
```
        image: chlam4/t8c-operator:7.21
```

3b. Edit the XL custom resource to add the desired annotation for the AWS probe which also requires 
a specially built image at the moment but those changes will become official soon.
```
  aws:
    enabled: true
  mediation-aws:
    annotations:
      "sidecar-injector-webhook.turbonomic.com/inject": "yes"
    image:
      repository: chlam4
      tag: 7.21.1-SNAPSHOT
      pullPolicy: Always
```

4. Verify sidecar container injected
```
$ kubectl -n turbonomic get pods | rg "aws-"
mediation-aws-65988685c9-tjs5m               2/2     Running   1          3h18m

$ kubectl -n turbonomic describe pod mediation-aws-65988685c9-tjs5m | egrep -i "(containers|annotations|mediation-aws|aws-secret|image):"
Annotations:    sidecar-injector-webhook.turbonomic.com/status: injected
Containers:
  mediation-aws:
    Image:          chlam4/com.vmturbo.mediation.aws.component:7.21.0-SNAPSHOT
  aws-secret-sidecar:
    Image:          chlam4/chris-poc-sidecar:v3
```
