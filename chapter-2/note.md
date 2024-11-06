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

### Creating a Subject

In the context of RBAC, you can use a user account, service account, or a group
as a subject. Users and Groups are not stored in etcd, the kubernetes database,
and arent meant for processes running outside of the cluster. Service Accounts
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
| authenticate             |

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

A user represetns a real person who commonly interacts with kubernetes using
kubectl.

A kuberentes cluster already comes with a ServiceAccount, the `default` service
account is used if no other ServiceAccount is referenced,and lives in the `default` namespace.

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

### Create A Role

to define new roles and rolebindings you will have to use a context that allows
for creating or midfying them, that is, cluster-admin or admin

##### Imperatively

```
$ kubectl craete role read-only --verb=list,get,watch \
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
