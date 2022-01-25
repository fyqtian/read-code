[在 Kubernetes 上安装 Gitlab CI Runner-阳明的博客|Kubernetes|Istio|Prometheus|Python|Golang|云原生 (qikqiak.com)](https://www.qikqiak.com/post/gitlab-runner-install-on-k8s/)

https://blog.csdn.net/cyfblog/article/details/100541760



[GitLab + Kubernetes: Running CI Runners in Kubernetes | Edenmal - Sysadmin Garden of Eden](https://edenmal.moe/post/2017/GitLab-Kubernetes-Running-CI-Runners-in-Kubernetes/?__cf_chl_managed_tk__=pmd_taZamjcjBHO2u.liIssylHCyrED0GG8FkTjiAVwPcO8-1631950603-0-gqNtZGzNAvujcnBszQjR)

```
helm install --namespace runner gitlab-runner -f <CONFIG_VALUES_FILE> gitlab/gitlab-runner

要修改gitlab 地址 token rbac

kubectl create namespace ops

helm install --namespace ops  gitlab-runer -f values.yaml gitlab/gitlab-runner


helm upgrade --namespace ops -f values.yaml gitlab-runer gitlab/gitlab-runner




 kubectl exec pod -n ops  -i -t -- sh
 
 
 
 
 
 kubectl dockerfile
 https://github.com/dtzar/helm-kubectl/blob/master/Dockerfile
 
 FROM alpine:3

ARG VCS_REF
ARG BUILD_DATE
ARG KUBE_VERSION
ARG HELM_VERSION

# Metadata
LABEL org.label-schema.vcs-ref=$VCS_REF \
      org.label-schema.name="helm-kubectl" \
      org.label-schema.url="https://hub.docker.com/r/dtzar/helm-kubectl/" \
      org.label-schema.vcs-url="https://github.com/dtzar/helm-kubectl" \
      org.label-schema.build-date=$BUILD_DATE

RUN apk add --no-cache ca-certificates bash git openssh curl gettext jq bind-tools \
    && wget -q https://storage.googleapis.com/kubernetes-release/release/v1.21.2/bin/linux/amd64/kubectl -O /usr/local/bin/kubectl \
    && chmod +x /usr/local/bin/kubectl \
    && wget -q https://get.helm.sh/helm-v3.6.3-linux-amd64.tar.gz -O - | tar -xzO linux-amd64/helm > /usr/local/bin/helm \
    && chmod +x /usr/local/bin/helm \
    && chmod g+rwx /root \
    && mkdir /config \
    && chmod g+rwx /config \
    && helm repo add "stable" "https://charts.helm.sh/stable" --force-update

WORKDIR /config

CMD bash
```

