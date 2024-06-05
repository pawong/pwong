---
author: ["Paul Wong"]
title: "Kubernetes Tricks"
date: "2023-01-18"
description: "Kubernetes Tips and Tricks"
summary: "This post has some useful Kubernetes tips and tricks"
tags: ["kubernetes", "devops", "snippet"]
categories: ["kubernetes", "snippet", "devops"]
ShowToc: true
TocOpen: true
---

[<img src="https://imgs.xkcd.com/comics/containers.png" alt="Kubernetes is hard">](https://xkcd.com/1988/)

## Dashboards

:::danger
Ask someone for the dashboard urls
:::

## kubectx & kubens

```bash
% brew install kubectx
```

### Switch Context and NameSpace

```bash
% kubectx # list contexts
docker-desktop
% kubectx --current # list current context
docker-desktop
% kubectx docker-desktop # set context
% kubens # list namespaces
default
kube-node-lease
kube-public
kube-system
postgres
% kubens --current # list current namespace
postgres
% kubens default # set namespace
% kubens - # set namespace to the prior one
```

## Connect to a pod

```bash
% kubectl exec --stdin --tty <pod-name> -- /bin/bash
```

## Other Commands

```bash
% kubectl get pods --all-namespaces
% kubectl config get-contexts
```

## Logs with kubetail

### Install

```bash
% asdf plugin-add kubetail https://github.com/janpieper/asdf-kubetail.git
% asdf install kubetail latest
% asdf global kubetail <version>
```

### Use

```bash
% kubetail <service name>
```

Or with a specific namespace

```bash
KUBETAIL_NAMESPACE=my-namespace kubetail <pod name>
```

## Kubectl useful commands

## List all objects

```
% kubectl get all -A
```

## Get object ID

```
% kubectl get namespace <namespace_name> --output jsonpath={.metadata.uid}
```

## List all services

```bash
% kubectl -n kube-system get services
```

or

```bash
% kubectl get services --all-namespaces
```

## View Config

```bash
% kubectl config view
```

## List and use contexts

```bash
% kubectl config get-contexts
% kubectl use-context <name>
```

## List and use namespaces

```bash
% kubectl get namespaces
% kubectl config set-context --current --namespace=<name>
```

## Pods

Get all pods

```bash
% kubectl get pods --namespace <namespace>
```

Describe pod

```bash
% kubectl describe pod <pod_name> --namespace <namespace>
```

Count pods

```
% k get pods | grep 'daemon' | wc -l
```

### Restart a Pod

You can use the following methods to `restart` a pod with kubectl. Once new pods are re-created they will have a different name than the old ones. A list of pods can be obtained using the kubectl get pods command.

```bash
% kubectl rollout restart
```

This method is the recommended first port of call as it will not introduce downtime as pods will be functioning. A rollout restart will kill one pod at a time, then new pods will be scaled up. This method can be used as of K8S v1.15.

```bash
% kubectl rollout restart deployment <deployment_name> -n <namespace>
```

## Services

Get services

```bash
% kubectl get svc
```

Describe service

```bash
% kubectl describe svc <service_name>
```

## Get a token

```bash
% kubectl create token default
```

## List Resources

```bash
% kubectl api-resources --namespaced=true
```

Specific resource get

```bash
% kubectl get secrets
```

## Delete Resources

Delete all resources.

```bash
% kubectl delete all --all -n {namespace}
```

The previous command doesn't delete admin level resource, using the following script will get rid of things such as limits, quotas, secrets, etc.

```bash
% kubectl delete "$(kubectl api-resources --namespaced=true --verbs=delete -o name | tr "\n" "," | sed -e 's/,$//')" --all
```

## Create Secret

```bash
% kubectl -n auth0-inter create secret generic auth0-inter-management-client-secret --from-literal=auth0__management_client_secret='fuck-you-ezekiel' --from-file secret/
```

## Get deployed image

```bash
% k get deployment -n <namespace> api -oyaml | grep -e "image:"
```

## Port Forwarding

```bash
% kubectl port-forward -n default service/halleck-kube-prometheus-stack-grafana <from_port>:<to_port
```

## Kubenetes with Docker Image

Build the image

```bash
% docker build -t jaikenone/fast-api .
```

Docker login and push

```bash
% docker login && docker push jaikenone/fast-api
```

Example deploy document, `deploy.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fast-api-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: fast-api
  template:
    metadata:
      labels:
        app: fast-api
    spec:
      containers:
        - name: fast-api
          image: bravinwasike/fast-api
          resources:
            limits:
              memory: "128Mi"
              cpu: "500m"
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: fast-api-service
spec:
  selector:
    app: fast-api
  ports:
    - port: 80
      targetPort: 8080
  type: LoadBalancer
```

Apply deploy document

```bash
kubectl apply -f deploy.yaml
```

## Create a Workstation Pod

ubuntu client

```
microk8s kubectl apply -f - <<EOF
---
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu
  namespace: default
spec:
  containers:
    - name: ubuntu
      image: ubuntu
      command:
        - sleep
        - infinity
EOF
```

## Connect

```
% kubectl exec -n default -it ubuntu -- /bin/bash
```

## How to delete Persistent Volume stuck in terminating state?

Change state of the PV

```
% kubectl patch pv <pv-name> -p '{"metadata":{"finalizers":null}}'
```

Delete PV

```
% kubectl delete pv <pv-name>
```
