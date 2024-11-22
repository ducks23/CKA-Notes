# Chapter 4. Scheduling and Tooling

Overview:

- Resource boundaries for Pods
- Imperative and declarative manifest management
- Common Templating tools like Kustomize, yq, and Helm.

## Understanding How Resource Limits Affect Pod Scheduling

A kubernetes cluster can consist of multiple Nodes. Deplending on the variety of rules, the kubernetes scheduler decides which node to pick for the running workload. The CKA exam doesn't ask you to understand the scheduling concepts mentioned previously, but it would be helpful to have a rough idea how they work at a high level

## Defining Container Resource Requests

Each contianer in a Pod can define its own resource Requests

| YAML Attribute                                         | Description             | Example Value       |
| :----------------------------------------------------- | :---------------------- | :------------------ |
| spec.conatainers[].resources.reques.cpu                | CPU Resource type       | 500m                |
| spec.containers[].resourcs.requests.memory             | Memory resource type    | 64Mi                |
| spec.containers[].resources.requess.hugepages-<size>   | Huge page resource type | hugepages-2Mi: 60Mi |
| spec.containers[].resources.requests.ephemeral-storage | ephemeral-storage type  | 4Gi                 |

Here is an example where we have two containers with their own resource
requests. A node that can run this Pod needs to have a minimum of 320Mi ram and 1250 CPU, the sum of both resource containers

Here is an example where we have two containers with their own resource
requests. A node that can run this Pod needs to have a minimum of 320Mi ram and 1250 CPU, the sum of both resource containers

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rate-limiter
spec:
  containers:
    - name: business-app
      image: bmuschko/nodejs-business-app:1.0.0
      ports:
        - containerPort: 8080
      resources:
        requests:
          memory: "256Mi"
          cpu: "1"
    - name: ambassador
      image: bmuschko/nodejs-ambassador:1.0.0
      ports:
        - containerPort: 8081
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
```

```bash
kubectl create -f rate-limiter-pod.yaml
```

## Defining Container Resource Limits

You can also set for a container resource limits: these ensure that the
container cannot consume more resources than is allotted to it.

| YAML Attribute                                       | Description             | Example Value       |
| :--------------------------------------------------- | :---------------------- | :------------------ |
| spec.conatainers[].resources.limits.cpu              | CPU Resource type       | 500m                |
| spec.containers[].resources.limits.memory            | Memory resource type    | 64Mi                |
| spec.containers[].resources.limits.hugepages-<size>  | Huge page resource type | hugepages-2Mi: 60Mi |
| spec.containers[].resources.limits.ephemeral-storage | ephemeral-storage type  | 4Gi                 |

In this example the container `business-app` cannot use more than 512Mi of
memory and 2000m of CPU. the container `ambassador` defines a lilmit of 128 Mi
of memory and 500m of CPU.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rate-limiter
spec:
  containers:
    - name: business-app
      image: bmuschko/nodejs-business-app:1.0.0
      ports:
        - containerPort: 8080
      resources:
        limits:
          memory: "512Mi"
          cpu: "2"
    - name: ambassador
      image: bmuschko/nodejs-ambassador:1.0.0
      ports:
        - containerPort: 8081
      resources:
        limits:
          memory: "128Mi"
          cpu: "500m"
```

### Recommended to Define Resource Requests and Limits

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rate-limiter
spec:
  containers:
    - name: business-app
      image: bmuschko/nodejs-business-app:1.0.0
      ports:
        - containerPort: 8080
      resources:
        requests:
          memory: "256Mi"
          cpu: "1"
        limits:
          memory: "512Mi"
          cpu: "2"
    - name: ambassador
      image: bmuschko/nodejs-ambassador:1.0.0
      ports:
        - containerPort: 8081
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
```

```bash
$ kubectl describe node minikube-m03
...
Non-terminated Pods:          (3 in total)
  Namespace                   Name                CPU Requests  CPU Limits   \
   Memory Requests  Memory Limits  AGE
  ---------                   ----                ------------  ----------   \
  ---------------  -------------  ---
  default                     rate-limiter        1250m (62%)   2500m (125%) \
  320Mi (14%)      640Mi (29%)    3s
...
```

## Declarative Object Management Using Kustomize

Supports 3 different use cases:

1. Generating manifests from other sources. For example creating a ConfigMap and
   populating its kv-pairs froma properties file

2. Adding common configuration across multiple manifests. For examlpe, adding a
   namespace and a set of labels for a Deployment and a Service.

3. Comnposing and customizing a collection of manifests. For example, setting
   resource boundaries for multiple deployments.

#### Render the produced output

This is like a dry run in kubectl

```bash
$ kubectl kustomize <target>
```

#### Creating the objects

This makes changes to the cluster

```bash
$ kubectl apply -k <target>
```
