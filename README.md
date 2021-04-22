# Reference Implementation: Kubernetes Relay Proxy 

This repository provides a set of configuration files that allow you to run the LaunchDarkly relay proxy as a single-node Kubernetes cluster locally. The yaml files provided in the `k8s` directory create a cluster enviornment with three instances of LD Relay and a shared Redis instance. The purpose of this project is to help LaunchDarkly employees familiarize themselves with the basics of Kubernetes, as well as provide a working cluster that can be used for troubleshooting or exploration.

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop): Allows you build and run containters
- [minikube](https://minikube.sigs.k8s.io/docs/start/): A tool that allows you to run a local Kubernetes cluster. Basically, a command line tool that interacts with a VM meant for running Kubernetes
- [kubectl](https://kubernetes.io/docs/tasks/tools/): Kubernetes command line tool, allows for container orchestration, 


## Getting Started

In the Terminal:

- `git clone https://github.com/SuperRockyCat/relay-kubernetes-reference && cd relay-kubernetes-reference`
- `minikube create`
- `minicube addons enable ingress`
- `kubectl apply -f k8s`

## Kubernetes Basics

"Kubernetes, also known as K8s, is an open-source system for automating deployment, scaling, and management of containerized applications." (https://kubernetes.io/) 

Many of our customers use Kubernetes in their production deployment workflows. As such, when customers are deploying the Relay Proxy for use with their LaunchDarkly SDKs, it's common for them to want to follow the same Kubernetes deployment pattern for ease of implementation and scaling. Kubernetes provides our customers with a centralized workflow for scaling, automating, and networking any piece of their application architecture.

### Kubernetes Terminology

_All definitions cited from https://kubernetes.io/docs/reference/glossary. Parenthetical text is added for clarity_ 

- **Container:** A lightweight and portable executable image that contains software and all of its dependencies.
- **Cluster:** A set of worker machines, called nodes, that run containerized applications. Every cluster has at least one worker node.
- **Node:** A worker machine in Kubernetes. (Could be a VM or Physical hardware)
- **Pod:** The smallest and simplest Kubernetes object. A Pod represents a set of running containers on your cluster. (Pods are generally only run one container, except in cases where there is a hard dependency between two containers- e.g. a frontend app and its logging mechanism)
- **Service:** An abstract way to expose an application running on a set of Pods as a network service.
- **Deployment:** An API object that manages a replicated application, typically by running Pods with no local state.
- **Object:** An entity in the Kubernetes system. The Kubernetes API uses these entities to represent the state of your cluster.

## Project Overview

Kubernetes has two approaches to interacting with its APIs: imperative and declarative. The imperative style makes use of the `kubectl` command line tool to interact with your clusters. It requires issuing terminal commands for each action you take. The declarative style, on the other hand, allows you to create configurations in the YAML format and apply them to your cluster using the `kubectl apply` command. This is much more in line with how Kubernetes is used in production settings, since it allows for easy change control of configuration changes. Additional reading available [here](https://medium.com/payscale-tech/imperative-vs-declarative-a-kubernetes-tutorial-4be66c5d8914)

The pods deployed in this cluster are configured by way of [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) objects. Any of the included files in the `k8s` directory containing the word deployment (relay-deployment.yaml, redis-deployment.yaml) determine the configuration of the pod. Let's take the `relay-deployment.yaml` for example, since it's the most relevant to our needs, and talk about what it does line by line:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: 
  name: relay-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      component: web
  template:
    metadata:
      labels:
        component: web
    spec:
      containers:
        - name: relay
          image: launchdarkly/ld-relay
          ports:
            - containerPort: 5000
          env:
            - name: USE_REDIS
              value: 'true'
            - name: REDIS_HOST
              value: redis-cluster-ip-service
            - name: ENV_DATASTORE_PREFIX
              value: ld-flags-$CID
            - name: LOG_LEVEL
              value: debug
            - name: TEST
              value: ok
            - name: AUTO_CONFIG_KEY
              value: rel-b0f30820-02d0-4c8c-82a2-1e911daedea1 # Replace with your own auto-config key
```

- The `apiVersion` field determines which Kubernetes API should be used when applying this configuration to an object. Pod configurations use the `apps/v1` api, however you will notice the `apiVersion` changes dependent on the object being configured. When in doubt, just Google the object `kind` + yaml + Kubernetes, and you should find the relevant doc explaining teh object's use.
- The `kind` field determines what object the configuration is creating
- The `metadata` field has a somewhat obvious use. In this case, we are only adding a name for our Pod.
- The `spec` field is where the pod specifications are determined. This contains information like the number of pod `replicas`, additional `metadata` like `labels` which are used to connect Services to pods (discussed later), as well as the configuration of the `container` deployed within that pod. At the bottom of the config, you can see what port we've assigned to the Relay Proxy, as well as the environment variables used to configure the relay proxy nested within the `env` field.

[More Reading on Services](https://kubernetes.io/docs/concepts/workloads/controllers/)

------

The other configs in the `k8s` directory create objects called Services and something called an Ingress.

**Services** are used to network pods within a cluster. Each deployment object has an associated Service which tells Kubernetes to expose all pods of a certain type (the `component` field nested in the `selector`) on a given port. This makes it easy for pods to communicate with one another, as you can see in the `relay-deployment.yaml` all that's required to point Relay at our Redis instance is to provide the name of the associated service: 

```yaml
  - name: REDIS_HOST
    value: redis-cluster-ip-service
```

[More reading on Services](https://kubernetes.io/docs/concepts/services-networking/service/)

------

An  **Ingress** is used to expose the Services within the cluster to the outside world. In this case, we are using an nginx based ingress called ingress-nginx. The Ingress config in this instance only needs to expose the relay service to the outside world, but as you can imagine the Ingress config can grow in complexity as the cluster does. The Ingress also does something different depending on the cloud provider it's deployed to. In some cases it spins up physical load balancer hardware with the relevant ingress config, in other is configues an application load balancer. This adds complexity when deploying to production.

[More reading on Ingresses](https://kubernetes.io/docs/concepts/services-networking/ingress/)
