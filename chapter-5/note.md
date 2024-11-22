# Chapter 5. Services and Networking

High level overview of Chapter:

- Kubernetes networking basics
- Conncetivity between Pods
- Services, service types and their endpoints
- Ingress controller and Ingress
- using and configuring CoreDns
- Choosing a container network interface plugin

## Kubernetes Networking basics

Kubernetes is designed as an operating system for managing the complexities of
distributed data and computing. Workloads can be scheduled on a set of nodes to
distribute the load. The kubernetes network model enables networking
communication and needs to fulfill the following requirements.

1. Container to container communication: Containers running in a Pod often need
   to communicate with eachother. Containers within the same Pods can send Inter
   Process Communication messages, share files, and most often communicate directly thorugh the loopback interface using the `localhost` hostname. Because
   each Pod is assigned a unique virtual IP address,each container in the same POd
   is given that context and shares the same port space.
2. Pod to Pod Communication: A pod needs to be able to reach another Pod running
   on the same or on a different node without NAT. Kubernetes assigns a unique
   IP address to every POd upon creation from the POd CIDR range of its node. The
   IP address is ephemeral and therefore cannot be considered stable over time.
   Every restart of a POd leases a new IP address. It's recommended to use
   Pod-to-service communcation over pod-to-pod communication.
3. Pod-to-servcie communication: Services expose a single, stable dns name for a
   set of pods with the capability of load balancing teh requests across the
   POds. Traffic to a service can be received from within the cluster or from the
   outside.
4. Node-to-node communication: Nodes registered with a cluster can talk to each
   other. Every node is assigned a node ip address.

## Connectivity bewtween containers

Containers created by a single pod share the same IP address and port spae.
Cotnainers can communicate among each other using `localhost`.

#### Multi-container pod

[app container] <---localhost---> [sidecar container]

This yaml creates a sidecar contaienr that calls the main application container
via `localhost:80`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container
spec:
  containers:
    - image: nginx
      name: app
      ports:
        - containerPort: 80
    - image: curlimages/curl:7.79.1
      name: sidecar
      args:
        - /bin/sh
        - -c
        - "while true; do curl localhost:80; sleep 5; done;"
```

#### Check to see successful by grabbing logs

```bash
$ kubectl create -f pod.yaml
pod/multi-container created
$ kubectl logs multi-container -c sidecar
...
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

## Connectivity Between Pods

Every pod is assigned an IP address upon creation. The IP address assigned to
a Pod is unique across all nodes and namespaces. This is achieved by assigning
to a dedicated subnet to each node when registering it.

It is important to understand that pods ip addresses are dynamic and get
reassigned when a pod is restarted or created again, so it is better to use a
`service` to communicate between containers in kubernetes.

## Understanding services

In a nutshell, SErvices provide discoverable names and load balancing to a set
of POds. The Services and Pods remain agnostic from IP addresses with the help
of kubernetes DNS control-plan component. Similar to a Deployment. the service
determines the pods it works on with the help label selection.

To acccess a service at a specific port you want traffic to reach the service at
port 80, then it is directed towards the target port which is the same port as
defined by the contianer running inside the label-selected Pod.

port 80
target port 8080 <-> container port 80808

## Access a SErvice with type ClusterIP

ClusterIP is the default type of service. It exposes the SErvice on a cluster
interanl IP address. That means the Service can accessed only from a Pod
ruunning inside of the cluster but not from outside of the cluster.

you cannot access this service from the node but you can reach it from inside
other pods

## Accessing a Service with Type NodePort

Declaring a Service with type NodePort exposes access thorugh the node's IP
address and can be resolved from outside of the kubernetes cluster. The node
