# Chapter 3 Workloads

When we talk about workloads in kubernets, we mean the API resource types that
run an application. Those include a Deployment, ReplicaSet, StatefulSet,
DaemonSet, Job, CronJob, and of course Pod.

### Concepts to Understand

- A basic understanding of Deployments
- Deployment rollout and rollback functionality
- Manual and automatic scaling of replicas controlled by a ReplicaSet
- configMap and secret

## Managing Workloads with Deployments

A workload is executed in a Pod.

Using a single instance of an application has a single point of failure since
all traffic is directed to that single instance. It is very problematic when
load indcreases due to higher demand.

### ReplicaSet

A kubernetes API resource that controls multiple, identical instances of a pod
running the application, so-called replicas. Has the ability of scaling the
number of replicas up or down on demand. Also knows how to roll out new
application over all the instances.

### Deployment

Abstracts the functionality of a ReplicaSet and manages it internally. In
practice this means that you don't have to manage the ReplicaSet yourself the
deployment keeps a history of the applications version and can roll back to an
older version to counteract blocking or potentially costly production issues.

### Creating Deployments

**imperatively**

```bash
$ kubectl create deployment app-cache --image=memcached:1.6.8 --replicas=4
deployment.apps/app-cache created
```

**declaritively**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-cache
  labels:
    app: app-cache # this label is irrelevant to the other two
spec:
  replicas: 4
  selector:
    matchLabels:
      app: app-cache # These need to match
  template:
    metadata:
      labels:
        app: app-cache # These need to match
    spec:
      containers:
        - name: memcached
          image: memcached:1.6.8
```
