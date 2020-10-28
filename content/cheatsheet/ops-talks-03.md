---
title: "Ops-Talks #03 - K8s Vs ECS"
aliases:
  - "ops-talks-03"
date: 2020-10-26T12:52:23+08:00
draft: false
tags: ["cheatsheet", "ops-talks-03", "k8s", "kubernetes", "helm", "ecs"]
categories: ["Syntax Highlighting"]
toc: false
type: "cheatsheet"
slide: https://slides.com/floriandambrine/ops-talks-03-k8s-vs-ecs
---

### 1.1 K8s Vs ECS

> Kubernetes is a portable, extensible, open-source platform for managing containerized workloads and services.
> **It is first and foremost a REST API**

* __Features:__ _(non-exhaustive)_
    * :small_blue_diamond: Service discovery and load balancing
    * :small_blue_diamond: Storage orchestration
    * :small_blue_diamond: Automated rollouts and rollbacks
    * :small_blue_diamond: Self-healing
    * :small_blue_diamond: Secret and configuration management
    * :small_blue_diamond: `kubectl` as a way to interact with the cluster

> ECS is a fully managed container orchestration service

* __Features:__
    * :small_orange_diamond: Integrated with AWS services
    * :small_orange_diamond: Easy to pick up / Entry level to container world
    * :small_orange_diamond: Ease of use from GUI + Serveless experience with Fargate
    * :small_orange_diamond: `ecs-cli` and `aws-cli` as a way to interact with the cluster

<!--more-->

### 1.2 Main concepts - Terminology

{{< figure src="/k8s-ecs-terms.png" class="center" caption="ECS Vs K8s - Terminology" width="50%">}}

* Terminology used in ECS and K8s world

| ECS                       | K8s              |
|---------------------------|------------------|
| Cluster                   | Cluster          |
| Service & Task definition | Deployment       |
| Task                      | Pod              |
| Volume                    | PersistentVolume |

---

### 2.1 Basic Navigation

* :rocket: **Setup for Kubernetes**

| Tool / Cli | Description |
| -- | --|
| [kubectl (EKS vendored)](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html) | Main Kubernetes Cli used to interact with the cluster |
| [kubens / kubectx](https://github.com/ahmetb/kubectx) | (Nice to have) Make it easy to switch clusters or switch between namespaces |
| [k9s](https://k9scli.io/) | K9s is a terminal based UI to interact with Kubernetes |

### 2.1.1 Authentication

* Grab EKS configuration

```bash {linenos=table,linenostart=1}
aws eks update-kubeconfig --name <CLUSTER_NAME_FROM_EKS> --alias <FRIENDLY_ALIAS>
```

* Switch to `verity-prod` cluster

```bash {linenos=table,linenostart=1}
#--- Using kubectx
kubectx verity-prod

#--- Native way
kubectl config set-context verity-prod \
  && kubectl config use-context verity-prod
```

* Switch to the namespace `default`

```bash {linenos=table,linenostart=1}
#--- Using kubens
kubens default

#--- Native way
kubectl config set-context verity-prod --namespace=default \
  && kubectl config use-context verity-prod
```

### 2.1.2 Navigate Clusters / Deployments / Pods / Specs

* These are  a really good resource, no need to bake one more :smiley:
    * [Denny Zhang - Kubernetes Cheatsheet](https://cheatsheet.dennyzhang.com/cheatsheet-kubernetes-A4) ([PDF version](https://github.com/dennyzhang/cheatsheet-kubernetes-A4/blob/master/cheatsheet-kubernetes-A4.pdf))
    * [Codefresh - Cheatsheet](https://codefresh.io/kubernetes-tutorial/kubernetes-cheat-sheet/)
    * [Official Kubernetes Cheatsheet Documentation](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

---

### 3.1 Deep Dive - Helm

> `Helm` is a package manager for kubernetes --  Helm is your new ecs-cli...

* A chart is a collection of files that describe a related set of Kubernetes resources.

* A chart is made of Go templates

* A single chart might be used to deploy something simple, like a memcached pod, or something complex, like a full web app stack with HTTP servers, databases, caches, and so on.

```bash {linenos=table,linenostart=1}
$ helm create ops-talks
  Creating ops-talks
```

* Layout of a chart:

```bash {linenos=table,linenostart=1}
# Tree of the created chart
$ tree ops-talks

ops-talks
├── Chart.yaml
├── charts
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml
```

* Helm charts can be fetched from open source git repositories, here are a few examples:
  * [helm install stable/<<chart>>](​https://github.com/helm/charts/)
  * [helm install cp-helm/<<chart>>](https://github.com/confluentinc/cp-helm)
  * [helm install jenkinsci/<<chart>>](https://github.com/jenkinsci/helm-charts)
  * [helm install lowess/<<chart>>](https://github.com/Lowess/helm-charts)

* `Helm` :potable_water: `hydrates` a Chart using values contained in `values.yaml`. Overrides can be defined allowing you to change the default behavior of the chart.


### 3.2 Deep Dive - Helmfile

> `Helmfile` is a wrapper on top of helm --  Helmfile is what Terragrunt is to Terraform...

* `Helm` is a great tool for **templating and sharing K8s manifests**...  However it can become quite cumbersome to install larger multi-tier applications or groups of applications across multiple Kubernetes clusters.

* Give each Helm chart its own `helmfile.yaml` and include them recursively in a **centralized** `helmfile.yaml`.

* Separate out environment specific values from general values. Often you’ll find while a Helm chart can take 50 different values, only a few actually differ between your environments.

* As well as providing a set of values, either Environment specific or otherwise, you can also read Environment Variables, Execute scripts and read their output (Fetch a secret from AWS SSM)

* Store remote state in git/s3/fileshare/etc in much the same way as Terraform does.

* `helmfile` project layout:

```bash {linenos=table,linenostart=1}
helmfile.yaml
environments.yaml
releases
├── kafka-lag-reporting
│   ├── README.md
│   ├── helmfile.yaml
│   └── values
│       ├── ai-kafka-lag-reporting--production.yaml
│       ├── ai-kafka-lag-reporting--staging.yaml
│       └── kafka-lag-reporting--production.yaml
├── kafka-manager
│   ├── helmfile.yaml
│   └── values
│       └── kafka-manager--production.yaml
├── prometheus
│   ├── helmfile.yaml
│   └── values
│       ├── prometheus-mle--production.yaml
│       ├── prometheus-nlp--production.yaml
│       └── prometheus-verity--production.yaml
└── zoonavigator
    ├── helmfile.yaml
    └── values
        └── verity-zoonavigator--production.yaml
```

* Sample content of an `helmfile.yaml`

```yaml {linenos=table,linenostart=1}
bases:
  - ../../environments.yaml

---

repositories:
  # Use Lowess (Florian Dambrine) OSS helm chart repo
  - name: lowess-helm
    url: https://lowess.github.io/helm-charts

templates:
  default: &default
    chart: "lowess-helm/karrot"
    missingFileHandler: Error
    namespace: "monitoring"
    labels: {}
    version: "0.1.3"
    wait: true
    installed: {{ and (env "KAFKA_LAG_REPORTING_INSTALLED" | default "true") }}

releases:
  - name: "ai-kafka-lag-reporting--{{ .Environment.Name }}"
    <<: *default
    values:
      - "./values/{{`{{ .Release.Name }}`}}.yaml"
```

### 3.3 GitOps

`GitOps` is a way to do Kubernetes cluster management and application delivery.  It works by **using Git as a single source of truth for declarative infrastructure and applications**.

With GitOps, the use of software agents can alert on any divergence between Git with what's running in a cluster, and if there's a difference, Kubernetes reconcilers automatically update or rollback the cluster depending on the case. With Git at the center of your delivery pipelines, developers use familiar tools to make pull requests to accelerate and simplify both application deployments and operations tasks to Kubernetes.

> Read more about GitOps on [Weaveworks blogpost](https://www.weave.works/technologies/gitops/)

![GitOps](https://images.contentstack.io/v3/assets/blt300387d93dabf50e/blt15812c9fe056ba3b/5ce4448f32fd88a3767ee9a3/what-is-gitops-2.png)

> #1 THE ENTIRE SYSTEM DESCRIBED DECLARATIVELY

> #2 THE CANONICAL DESIRED SYSTEM STATE VERSIONED IN GIT

> #3 APPROVED CHANGES THAT CAN BE AUTOMATICALLY APPLIED TO THE SYSTEM

> #4 SOFTWARE AGENTS TO ENSURE CORRECTNESS AND ALERT ON DIVERGENCE.
