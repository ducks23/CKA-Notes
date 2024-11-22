# Chapter 2 Cluster Architecture, Installation, and Configuration

#### Summary

- Understanding RBAC
- Installation of a cluster with `kubeadm`
- Upgrading a version of a cluster with `kubeadm`
- Backing up and restoring etcd with etcdctl
- understanding a highly available kubernetes cluster

## Role Based Access Control

RBAC helps with implementing a variety of use cases:

- Establishing a system for users with different roles to access a set of
  Kubernetes resources
- Controlling a process running in a Pod and the operations they can perform
  via the Kubernetes API
- Limiting the visibility of certain resources per namespace

_RBAC_ consists of three key building blocks and together, they connect API
primitives and thier allowed operations to the so-called subject, which is a
user, a group, or a ServiceAccount.

#### Subject

The user or process that wants access to a resources

#### Resource

The Kubernetes API resource type(eg., a deployment or node)

#### Verb

The operation that can be executed on the resource (creating a pod or deleting a
service)

---

### Creating a Subject

In the context of RBAC, you can use a user account, service account, or a group
as a subject. Users and Groups are not stored in etcd, the kubernetes database,
and aren't meant for processes running outside of the cluster. Service Accounts
exist as objects in kubernetes and are use by processes running inside the
cluster

Kubernetes does not represent a user as with an API Resource. The user is meant
to be managed by the admin of a kubernetes cluster, who then distributes
credentials to the real person.

| Authentication Strategy  | Description                                              |
| :----------------------- | :------------------------------------------------------- |
| x.509 client certificate | Uses an OpenSSL client cert to authenticate              |
| Basic Authentication     | Uses username and password to authenticate               |
| Bearer Token             | USes openID (a flavor of oauth2) or webhooks as a way to |

#### How to use OpenSSL client certificate to authenticate

1. Log into the kubernetes control plan node and create a temporary directory
   that will hold generated keys:
   ```
   $ mkdir cert && cd cert
   ```
2. Create a private key using the openssl executable. Provide an expressive file
   name, such as <username>.key:

   ```
   $ openssl genrsa -out johndoe.key 2048
   Generating RSA private key, 2048 bit long modulus
   ..............................+
   ..+
   e is 65537 (0x10001)
   $ ls
   johndoe.key
   ```

3. Create a ceritifcate sign request (CSR) file. Provide private key from prev
   step.
   ```
   $ openssl req -new -key johndoe.key -out johndoe.csr -subj \
   "/CN=johndoe/O=cka-study-guide"
   $ ls
   johndoe.csr johndoe.key
   ```
4. Sign CSR with the kuberentes cluster certificate authority (CA). The CA can
   usually be found the directory `/etc/kubernetes/pki` and needs to contain the
   files `ca.crt` and `ca.key`. We are using minikube so its a bit different.

   ```
   openssl x509 -req -in johndoe.csr -CA /.minikube/ca.crt -CAkey \
   /.minikube/ca.key -CAcreateserial -out johndoe.crt -days 364
   Signature oksubject
   subject=/CN=johndoe/O=cka-study-guideGetting
   Getting CA Private Key
   ```

5. Create the user in kuberentes by setting a user entry in kubeconfig for
   johndoe. Point to the CRT and key file. Set a context entry in kubeconfig for
   johndoe:

   ```
   $ kubectl config set-credentials johndoe \
    --client-certificate=johndoe.crt --client-key=johndoe.key
   User "johndoe" set.

   $ kubectl config set-context johndoe-context --cluster=minikube \
      --user=johndoe
   Context "johndoe-context" modified.
   ```

6. To switch to the user, use the context named `johndoe-context`.
   ```
    $ kubectl config use-context johndoe-context
    Switched to context "johndoe-context".
    $ kubectl config current-context
    johndoe-context
   ```

### Service Account

> A user represents a real person who commonly interacts with kubernetes using
> kubectl. You can assign a service account to a pod for example if a pod is
> running the service `helm` and needs to interact with the `kubectl` api.
>
> A kuberentes cluster already comes with a ServiceAccount, the `default` service
> account is used if no other ServiceAccount is referenced,and lives in the `default` namespace.

To create a customer ServiceAccount imperatively, run:

```
$ kubectl create serviceaccount build-bot
serviceaccount/build-bot created
```

can also write it declaritvely with:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-bot
```

#### list service Accounts

```
$ kubectl get serviceaccounts
NAME        SECRETS   AGE
build-bot   1         78s
default     1         93d
```

#### Get ServiceAccountDetails:

Renders additonal information stored about service account.
You can see it stores a secret token as well.

```
$ kubectl describe serviceaccount build-bot
Name:                build-bot
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   build-bot-token-rvjnz
Tokens:              build-bot-token-rvjnz
Events:              <none>
```

### Assign a ServiceAccount to a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: build-observer
spec:
  serviceAccountName: build-bot
```

## Understanding RBAC API Primitives

#### Functionality

##### Role

> The Role API primitives declare the api resources and their operations
> this rule should operate on. For ex. you may want to say "allow listing and
> deleting of pods" or you may express "allow watching the logs of Pods," or
> even both with the same role. Any operation that is not spelled out
> explicitly is disallowed as soon as it is bound to the subject

##### RoleBinding

> The RoleBinding API primitive binds the Role object to the subjects. It is the
> glue for making the rules active. For ex., you may want to say "bind the role
> that permits updating Services to the user John Doe"

### Create A Role

to define new roles and rolebindings you will have to use a context that allows
for creating or modifying them, that is, cluster-admin or admin.

##### Imperatively

```
$ kubectl create role read-only --verb=list,get,watch \
  --resource=pods,deployments,services
role.rbac.authorization.k8s.io/read-only created
```

##### Declaritvely

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: read-only
rules:
  - apiGroups:
      - ""
    resources:
      - pods
      - services
    verbs:
      - list
      - get
      - watch
  - apiGroups:
      - apps
    resources:
      - deployments
    verbs:
      - list
      - get
      - watch
```

##### Listing roles

```
$ kubectl get roles
NAME        CREATED AT
read-only   2021-06-23T19:46:48Z
```

##### Desribe roles

```
$ kubectl describe role read-only
Name:         read-only
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources         Non-Resource URLs  Resource Names  Verbs
  ---------         -----------------  --------------  -----
  pods              []                 []              [list get watch]
  services          []                 []              [list get watch]
  deployments.apps  []                 []              [list get watch]

```

## Creating RoleBindings

Once you have a role created with specific verbs allowed to perform actions on
certain objects you have to create a rolebinding to assign the role to a
ServiceAccount.

#### Imperatively

```
$ kubectl create rolebinding read-only-binding --role=read-only --user=johndoe
rolebinding.rbac.authorization.k8s.io/read-only-binding created
```

#### Declaritvely

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-only-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: read-only
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: johndoe
```

#### Get Role Bindings

```
$ kubectl get rolebindings

```

#### Render RoleBinding Details

```
$ kubectl describe rolebinding read-only-binding
```

## furthermore

We have created a user that is read only now if we create a deployment with the
default context but switch to user john doe, that user will not be able to get
replicasets or delete deployments.

You can verify your permissions with the `can-i` command

```
$ kubectl auth can-i --list --as johndoe
Resources          Non-Resource URLs   Resource Names   Verbs
...
pods               []                  []               [list get watch]
services           []                  []               [list get watch]
deployments.apps   []                  []               [list get watch]
$ kubectl auth can-i list pods --as johndoe
yes

```

### Namespace and Cluster wide RBAC

Roles and Rolebindings apply to a particular namespace. You will have to
specifiy the namespace at the time of creating both objects. Sometimes, a set of
Roles and Rolebindings needs to apply to multiple namespaces or even the whole
cluster. Kubernetes API offers the API resources types `ClusterRole` and
`ClusterRoleBinding`.

#### Aggregating RBAC Rules

You can declaritvely apply the cluster role bindings to a namespace and even add
them together to make a single clusterRole using the aggregation of multiple
rules

for example we have two roles, list pods and delete services.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: list-pods
  namespace: rbac-example
  labels:
    rbac-pod-list: "true"
rules:
  - apiGroups:
      - ""
    resources:
      - pods
    verbs:
      - list

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: delete-services
  namespace: rbac-example
  labels:
    rbac-service-delete: "true"
rules:
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - delete
```

To Aggregate these rules clusterRoles can specify an aggregationRule.

```yaml
kind: ClusterRole
metadata:
  name: pods-services-aggregation-rules
  namespace: rbac-example
aggregationRule:
  clusterRoleSelectors:
    - matchLabels:
        rbac-pod-list: "true"
    - matchLabels:
        rbac-service-delete: "true"
rules: []
```

# Configuration and Installation

> You can always retrieve the join command by running `kubeadm token create --print-join-command` on the control plane node should you lose it.

1. Boostrapping a control plane node

   1. ssh into machine
   2. kubeadm init

   ```bash
      $ kubeadm init --pod-network-cidr 172.18.0.0/16 \
        --api-server-advertise-addresss 10.8.8.10
      ...
      To start using your cluster, you need to run the following as a regular user:

        mkdir -p $HOME/.kube
        sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        sudo chown $(id -u):$(id -g) $HOME/.kube/config

      You should now deploy a pod network to the cluster.
      Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
        https://kubernetes.io/docs/concepts/cluster-administration/addons/

      Then you can join any number of worker nodes by running the following on \
      each as root:

      kubeadm join 10.8.8.10:6443 --token fi8io0.dtkzsy9kws56dmsp \
          --discovery-token-ca-cert-hash \
          sha256:cc89ea1f82d5ec460e21b69476e0c052d691d0c52cce83fbd7e403559c1ebdac
   ```

   3. copy kubeconfig for non-root users
   4. install cni plugin
      - `kubectl apply -f cni-plugin.yaml`
   5. check node status
      - `kubectl get nodes -o wide`

2. set up worker node

   1. ssh into machine
   2. kubeadm join

   ```
   $ sudo kubeadm join 10.8.8.10:6443 --token fi8io0.dtkzsy9kws56dmsp \
      --discovery-token-ca-cert-hash \
      sha256:cc89ea1f82d5ec460e21b69476e0c052d691d0c52cce83fbd7e403559c1ebdac
    [preflight] Running pre-flight checks
    [preflight] Reading configuration from the cluster...
    [preflight] FYI: You can look at this config file with \
    'kubectl -n kube-system get cm kubeadm-config -o yaml'
    [kubelet-start] Writing kubelet configuration to file \
    "/var/lib/kubelet/config.yaml"
    [kubelet-start] Writing kubelet environment file with \
    flags to file "/var/lib/kubelet/kubeadm-flags.env"
    [kubelet-start] Starting the kubelet
    [kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

    This node has joined the cluster:
    * Certificate signing request was sent to apiserver and a response was received.
    * The Kubelet was informed of the new secure connection details.

    Run 'kubectl get nodes' on the control plane to see this node join the cluster.
   ```

   3. (optional) scp kubeconfig
   4. check node status
   5. exit machine

## HA Cluster

Setting up a HA CLuster means setting up a cluster with multiple control plane
nodes if one of them goes down. If you lose your control plane node you would
lose all ability to use the kubectl API.

#### `Stacked ETCD Topology`

involves creating two or more control plane nodes where etcd is colocated on the control plane nodes. Each controle plane node hosts
the api server, the scheduler, and the controller manager. Worker nodes
communicate with the api server through a load balancer. It is recommended to
operate the cluster with a minimum of three control plane nodes for redundancy
reasons.

#### `The External ETCD Node Topology`

seperates etcd from the control plane node by running it on a dedicated machine.
