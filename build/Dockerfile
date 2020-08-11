FROM alpine:latest

# set labels for metadata
LABEL maintainer="Cheuk Lam<cheuk.lam@turbonomic.com>" \
  name="k8s-secret-injector" \
  description="A Kubernetes mutating webhook server that implements injection of kubernetes secret as volume" \
  summary="A Kubernetes mutating webhook server that implements k8s secret injection"

# set environment variables
ENV K8S_SECRET_INJECTOR=/usr/local/bin/k8s-secret-injector \
  USER_UID=1001 \
  USER_NAME=k8s-secret-injector

# install k8s-secret-injector binary
COPY build/_output/bin/k8s-secret-injector ${K8S_SECRET_INJECTOR}

# copy licenses
RUN mkdir /licenses
COPY LICENSE /licenses

# set entrypoint
ENTRYPOINT ["/usr/local/bin/k8s-secret-injector"]

# switch to non-root user
USER ${USER_UID}
