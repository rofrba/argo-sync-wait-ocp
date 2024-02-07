FROM registry.redhat.io/openshift4/ose-cli:latest

USER root

RUN curl -sSL -o /usr/local/bin/argocd \ 
    https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64 && \
    chmod +x /usr/local/bin/argocd

RUN mkdir -p /.kube && chmod -R 775 /.kube

RUN mkdir -p /.config && chmod -R 775 /.config

USER 1001

