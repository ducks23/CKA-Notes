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

### Get Info

```bash
$ kubectl get deployments
$ kubectl get pods
$ kubectl kubectl describe deployment app-cache
```

### Delete a deployment

```bashrc
$ kubectl get deployments,pods,replicasets
$ kubectl delete deployment app-cache
deployment.apps "app-cache" deleted
$ kubectl get deployments,pods,replicasets
No resources found in default namespace
```

## Performing Rolling Upadates and Rollbacks

### Rolling Out a new version of application

You just need to change the desired state of the pod template and the deployment
takes care of updating all replicas to the new version one by one. This process
is called a _rolling update_.

```bash
$ kubectl set image deployment app-cache memcached=memcached:1.6.10 --record
```

the flag --record is optional and defaults to `false` if provided without a
value or set to the value true, the command used for the change will be
recorded.

You can check the status of the rollout

```bash
$ kubectl rollout status deployment app-cache
Waiting for rollout to finish: 2 out of 4 new replicas have been updated...
deployment "app-cache" successfully rolled out
```

Check the history of deployments

```
$ kubectl rollout history deployment app-cache
deployment.apps/app-cache
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image deployment app-cache memcached=memcached:1.6.10 \
          --record=true

```

to get a more detailed view of a specific revision run

```bash
$ kubectl rollout history deployments app-cache --revision=2
deployment.apps/app-cache with revision #2
Pod Template:
  Labels:   app=app-cache
   pod-template-hash=596bc5586d
  Annotations: kubernetes.io/change-cause: kubectl set image deployment \
                app-cache memcached=memcached:1.6.10 --record=true
  Containers:
   memcached:
    Image:  memcached:1.6.10
    Port:   <none>
    Host Port: <none>
    Environment:  <none>
    Mounts: <none>
  Volumes:  <none>
```

### Rolling back to a previous version

You can pick a specific revision to roll back to with just one command.

```
kubectl rollout undo deployment app-cache --to-revision=1
```

## Scalilng Workloads

Scalabilit is one of kuberentes' build in capabilities. We'll learn how
tomanually scale the number of replicas as a reaction to increased load on the
application. Furhtermore, we'll talk about the API resource Horizontal Pod
Autoscaler, which allows to automatically scale the managed set of Pods based on
resource thresholds liek CPU and Memory.

### Manually Scaling a Deployment.

very simple you just change the desired amount of replicas you wish to scale to.

```bash
$ kubectl scale deployment app-cache --replicas=6
deployment.apps/app-cache scaled

$ kubectl get pods
NAME                         READY   STATUS    RESTARTS   AGE
app-cache-5d6748d8b9-6cc4j   1/1     Running   0          3m17s
app-cache-5d6748d8b9-6rmlj   1/1     Running   0          32m
app-cache-5d6748d8b9-6z7g5   1/1     Running   0          3m17s
app-cache-5d6748d8b9-96dzf   1/1     Running   0          32m
app-cache-5d6748d8b9-jkjsv   1/1     Running   0          31m
app-cache-5d6748d8b9-svrxw   1/1     Running   0          32m
```

Additionally you can scale StatefulSets

```bash
$ kubectl scale statefulset redis --replicas=3
statefulset.apps/redis scaled
$ kubectl get statefulset redis
NAME    READY   AGE
redis   3/3     3m43s
$ kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
redis-0   1/1     Running   0          101m
redis-1   1/1     Running   0          97m
redis-2   1/1     Running   0          97m
```

### Autoscaling a Deployment

Another way to scale a deployment is with the help of a horizontal Pod
Autoscaler. HPA is an API primitive that defines rules for automatically scaling
the number of replicas under certain conditions like cpu utilization.

#### Imperatively

```bash
$ kubectl autoscale deployment app-cache --cpu-percent=80 --min=3 --max-5
$ kubectl get hpa
NAME        REFERENCE              TARGETS         MINPODS   MAXPODS   REPLICAS \
  AGE
app-cache   Deployment/app-cache   <unknown>/80%   3         5         4        \
  58s
```

#### Declaritively

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: app-cache
spec:
  maxReplicas: 5
  minReplicas: 3
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app-cache
  targetCPUUtilizationPercentage: 80
```

note that the target might be set to unkown because you have not declaritively
defined the cpu usage limits inside the deployment yaml. We will need to update
it by adding resource limits

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
          resources:
            requests:
              cpu: 250m
            limits:
              cpu: 500m
```

After Updating that deployment we should see the target show the percentage of
resources used

```bash
$ kubectl get hpa
NAME        REFERENCE              TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
app-cache   Deployment/app-cache   15%/80%   3         5         4          58s
```

Although that is the older version of the autoscaling API, you can define it
better using the `autoscaling/v2` API

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-cache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app-cache
  minReplicas: 3
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 80
    - type: Resource
      resource:
        name: memory
        target:
          type: AverageValue
          averageValue: 500Mi
```

# Defining and Consuming Configuration Data

At times we want to put environment variables in our containers we can declare
environment variables very easily by listing them as key-value pairs under the
attribute `spec.containers[].env[]`. This ecample shows the variables
`MEMCACHED_CONNCECTIONS` and `MEMCACHED_THREADS`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memcached
spec:
  containers:
    - name: memcached
      image: memcached:1.6.8
      env:
        - name: MEMCACHED_CONNECTIONS
          value: "2048"
        - name: MEMCACHED_THREADS
          value: "150"
```

## Creating a configMap

You can create a ConfigMap by emitting the imperative `create configmap`. This
command requires you to provide the source of data as an option.

| Option             | Example                       | Description                                                                  |
| ------------------ | ----------------------------- | ---------------------------------------------------------------------------- |
| `--from-literal`   | `--from-literal=locale=en_US` | Literal Values, which are key-value pairs as plain text                      |
| `--from-env-files` | `--from-env-file=config.env`  | a file that contains kv pairs and expectems them to be environemnt variables |
| `--from-file`      | `--from-file=app-config.json` | A file with arbitrary contents                                               |
| `--from-file`      | `--from-file=config-dir`      | a dir with one or many files                                                 |

#### Literal Example

```bash
$ kubectl create configmap db-config --from-literal=DB_HOST=mysql-service
--from-literal=DB_USER=backend

```

#### declaritive Example

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
data:
  DB_HOST: mysql-service
  DB_USER: backend
```

## Consuming a ConfigMap as Environment Variables

Now that we have that configMap defined in the cluster we can reference the
config map simple from referencing the name

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend
spec:
  containers:
    - image: bmuschko/web-app:1.0.1
      name: backend
      envFrom:
        - configMapRef:
            name: db-config # referenced here.
```

## Mounting a ConfigMap as a Volume.

```json
{
  "db": {
    "host": "mysql-service",
    "user": "backend"
  }
}
```

```bash
$ kubectl create configmap db-config --from-file=db.json
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend
spec:
  containers:
    - image: bmuschko/web-app:1.0.1
      name: backend
      volumeMounts:
        - name: db-config-volume
          mountPath: /etc/config
  volumes:
    - name: db-config-volume
      configMap:
        name: db-config
```

## Creating Secrets

Secrets need to be base 64 encoded before use, so make sure to do it yourself
using the `base64` unix command.

How its done:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-basic-auth
type: kubernetes.io/basic-auth
stringData:
  username: bmuschko
  password: secret
```

Then you can inject them into your container here.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend
spec:
  containers:
    - image: bmuschko/web-app:1.0.1
      name: backend
      envFrom:
        - secretRef:
            name: secret-basic-auth
```

## Mounting SSH Secret as Volume.

You can mount your ssh key in `~/.ssh/id_rsa` in a pod by creating a secret in
the cluster and referencing it as a volume in your pod.

```bash
$ cp ~/.ssh/id_rsa ssh-privatekey
$ kubectl create secret generic secret-ssh-auth --from-file=ssh-privatekey \
  --type=kubernetes.io/ssh-auth
secret/secret-ssh-auth created
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend
spec:
  containers:
    - image: bmuschko/web-app:1.0.1
      name: backend
      volumeMounts:
        - name: ssh-volume
          mountPath: /var/app
          readOnly: true
  volumes:
    - name: ssh-volume
      secret:
        secretName: secret-ssh-auth
```

```bash
kubectl exec -it backend -- /bin/sh
# ls -1 /var/app
ssh-privatekey
# cat /var/app/ssh-privatekey
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,8734C9153079F2E8497C8075289EBBF1
...
-----END RSA PRIVATE KEY-----
```
