# Chapter 1

## Exam ObjectiveA

kubernetes clusters need to be installed, configured, and maintained by skilled
professionals. That's the job of a kubernetes admin.

## Curriculum

high level overview of the sections of the test.

- 25% Cluster Architecture, installation, and configuration
- 15% Workloads and Scheduling
- 20% Services and Networking
- 10% Storage
- 30% Troubleshooting

[insert screenshot here]

[FAQ](https://docs.linuxfoundation.org/tc-docs/certification/faq-cka-ckad-cks) for exam

## Time management

Candidates have a time limit of two hours to complete the exam. At a minimum, 66% of the answers to the questions need to be correct.

## Command Line Tips and Tricks

Setting Context and Namespace

```bash
$ kubectl config set-context <context-of-question> \
  --namespace=<namespace-of-question>
$ kubectl config use-context <context-of-question>
```

Setting Alias

```bash
$ alias k=kubectl
$ k version
```

Command API Resource lists all available commands plus their short names:

```bash
$ kubectl api-resources
NAME                    SHORTNAMES  APIGROUP  NAMESPACED  KIND
...
persistentvolumeclaims  pvc                   true        PersistentVolumeClaim
...
```

You may want to delete objects quicker. Kubernetes may take 30 seconds to
gracefully delete something

To delete it by force:

```bash
$ kubectl delete pod nginx --force
```

## Finding Object information

May need to find information about the current system for example

```bash
kubectl describe pods | grep -C 10 "author=John Doe"
$ kubectl get pods -o yaml | grep -C 5 labels:
```

This finds all pods with the annotation key-value pair author=John Doe plus the
surrounding 10 lines. The second command searches the YAML representation of all
pods for their labels including the surrounding five lines of output.

### Discover Command Options

```bash
Create a resource from a file or from stdin.

JSON and YAML formats are accepted.

Examples:
  ...

Available Commands:
  ...

Options:
  ...

```

You may also get more specific by adding the json path to the object.

```bash
$ kubectl explain pods.spec

KIND:     Pod
VERSION:  v1

RESOURCE: spec <Object>

DESCRIPTION:
  ...

FIELDS:
  ...

```
