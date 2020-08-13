# Kubernetes Mutating Webhook for Sidecar Injection

[![GoDoc](https://godoc.org/github.com/chlam4/kube-mutating-webhook-tutorial?status.svg)](https://godoc.org/github.com/chlam4/kube-mutating-webhook-tutorial)

This shows how to build and deploy a [MutatingAdmissionWebhook](https://kubernetes.io/docs/admin/admission-controllers/#mutatingadmissionwebhook-beta-in-19) that injects a Kubernetes secret as a volume into pods with corresponding annotations.

# Webhook Deployment

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

The k8s secret inject webhook should be running
```
$ kubectl -n k8s-secret-injector get pods
NAME                                                     READY   STATUS    RESTARTS   AGE
k8s-secret-injector-webhook-deployment-dbfd96b7f-nfjqr   1/1     Running   0          2m24s
$ kubectl -n k8s-secret-injector get deployment
NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
k8s-secret-injector-webhook-deployment   1/1     1            1           51m
```

# Secret Injection Annotations

This section describes how to annotate the service deployments to inject db credentials from Kubernetes secrets.

## Label
Label the desired namespace with `k8s-secret-injector=enabled`.  In the example below, we use `turbonomic` as the namespace.
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

## Annotate
Annotate the turbonomic Custom Resource (CR) for desired components to mount secrets as a volume.

We currently support the following two kinds of annotations:
* If a component has its own credentials, then use this annotation to specify the specific secret name 
(`foo` in the example below):
```
  action-orchestrator:
    annotations:
      k8s-secret-injector-webhook.turbonomic.com/inject: "true"
      k8s-secret-injector-webhook.turbonomic.com/secret: "foo"
```
* If no secret name is specified in the annotation like the following, then a default secret named `mariadb` is assumed. 
```
  action-orchestrator:
    annotations:
      k8s-secret-injector-webhook.turbonomic.com/inject: "true"
```

## Verify
To verify, please check the component's pod spec for the `volumeMount` and `volumes` sections to confirm that 
the Kubernetes secret has been injected as a volume.  In the following example, the secret named `foo` has been 
injected and mounted.  The mountpath is `/vault/secrets`, just because Turbonomic also supports secret injection from 
Hashicorp vault. 
```
spec:
  containers:
  - volumeMounts:
    - mountPath: /vault/secrets
      name: db-creds
  volumes:
  - name: db-creds
    secret:
      defaultMode: 420
      optional: true
      secretName: foo
```
## Troubleshooting

If the pod isn't created or its spec isn't mutated properly, check the following items:

1. The webhook is in running state and no error logs.
2. The namespace in which application pod is deployed has the correct labels as configured in `mutatingwebhookconfiguration`.
3. Check the `caBundle` is patched to `mutatingwebhookconfiguration` object by checking if `caBundle` fields is empty.
4. Check if the application pod has annotation `k8s-secret-injector-webhook.turbonomic.com/inject":"yes"`.

# Credentials Secret Creation

To create the default secret (name `mariadb`) in namespace `turbonomic` with username `db_user` and password 
`db_password`:
```
cat <<EOF | kubectl -n turbonomic apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: mariadb
type: Opaque
stringData:
  db-creds: |-
    username: 'db_user'
    password: 'db_password'
```
To create a specific secret `foo` with the same set of credentials:
```
cat <<EOF | kubectl -n turbonomic apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: foo
type: Opaque
stringData:
  db-creds: |-
    username: 'db_user'
    password: 'db_password'
```

To verify that the credentials are mounted properly inside the component container:
```
$ kubectl -n turbonomic exec -it action-orchestrator-6869ddb975-64nc4 -- cat /vault/secrets/db-creds
username: 'db_user'
password: 'db_password'%
```

As a reference, here are the MySQL commands to create the above user and grant privileges on the `action` database, 
after logging on as a user with sufficient privileges:
```
CREATE USER 'db_user'@'%' IDENTIFIED BY 'db_password';
GRANT ALL PRIVILEGES ON action.* TO 'db_user'@'%';
```