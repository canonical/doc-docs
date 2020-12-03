[comment]: <> (How To Perform Maintenance On A Kubernetes Worker)

## Introduction

Duration: 20:00

Often times, work will need to be done on a Kubernetes worker or the
underlying hypervisor. This tutorial shows how to stop Kubernetes worker unit
and then bring it back online.

### Requirements

To make your way through this tutorial, you will need:

- Juju installed (see [Installing Juju](https://jaas.ai/docs/installing))
- Juju controller up and running (see
[Creating a controller](https://juju.is/docs/creating-a-controller))
- Kubectl command line utility installed (see
[kubectl snap](https://snapcraft.io/kubectl))

### Environment preparation

In case that you already have a juju model running a kubernetes cluster with
multiple workers, you can skip this section.

This tutorial can be completed with either [Charmed Kubernetes](https://jaas.ai/canonical-kubernetes)
or with [Kubernetes Core](https://jaas.ai/kubernetes-core). For testing
purposes, Kubernetes Core is enough.

Start by deploying Kubernetes Core bundle and increasing the number of
Kubernetes Workers.

```console
$ juju deploy cs:bundle/kubernetes-core
$ juju add-unit kubernetes-worker
```

Wait for kubernetes deployment to settle down, occasionaly running
`juju status` until every unit doesn't reach the `active/idle` state. Once the
deployment settles, you can download kubectl configuration file from the
`kubernetes-master` unit.

```console
$ juju scp kubernetes-master/0:config ~/.kube/config
```

#### (Optional) Workload deployment

For better demostration of pod migration during the maintenance we can deploy
dummy workload on the Kubernetes cluster. Create `nginx-deployment.yaml` file
with following content.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 4
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

Deploy it by running:

```console
$ kubectl apply -f ./nginx-deployment.yaml
```

This will start 4 pods with nginx image that should be evenly deployed on
available kubernetes-worker units.

### Testing install

Run following command to verify that we have multiple kubernetes nodes. Each
node represents one kubernetes-worker unit.

```console
$ kubectl get nodes
NAME                STATUS   ROLES    AGE    VERSION
juju-37550a-k8s-1   Ready    <none>   10h    v1.19.4
juju-37550a-k8s-3   Ready    <none>   140m   v1.19.4
```

If you have workload deployed on the kubernetes cluster, you can check
distribution of your pods among the nodes by running.

```console
kubectl get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP            NODE                NOMINATED NODE   READINESS GATES
nginx-deployment-7848d4b86f-9zcl2   1/1     Running   0          25s   10.1.94.13    juju-37550a-k8s-3   <none>           <none>
nginx-deployment-7848d4b86f-hpjqv   1/1     Running   0          25s   10.1.100.13   juju-37550a-k8s-1   <none>           <none>
nginx-deployment-7848d4b86f-pzqkt   1/1     Running   0          25s   10.1.94.14    juju-37550a-k8s-3   <none>           <none>
nginx-deployment-7848d4b86f-tn7bf   1/1     Running   0          25s   10.1.100.12   juju-37550a-k8s-1   <none>           <none>
```

[note status="What to do if you get stuck"]
The best place to ask for help is the [Charmhub Discourse forum](https://discourse.charmhub.io/).

If you prefer chatting, visit us on [IRC](https://webchat.freenode.net/#juju).
[/note]

## Draining Kubernetes Worker

Duration: 10:00

The kubernetes-worker charm provides actions for draining workers. These
actions are `pause` and `resume`. Behind the scenes, `pause` runs a kubectl
command to drain the node of all pods running on it. These pods will be
rescheduled onto other nodes.

First, check to make sure there's enough resource capacity in the cluster to
handle taking out a node.

```console
$ kubectl top nodes
NAME                CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
juju-37550a-k8s-1   1121m        28%    1476Mi          21%
juju-37550a-k8s-3   1185m        29%    1458Mi          21%
```

Then you can drain node of your choice identified by unit ID. To drain
`kubernetes-worker/0` run following.

```console
$ juju run-action --wait kubernetes-worker/0 pause
```

If the action fails with error regarding local storage

```plain
error: cannot delete Pods with local storage (use --delete-local-data to
override)
```

You will need to re-run the action and pass the `--delete-local-data=true`
argument. Be warned that this will cause loss of localy stored data in pods
that use `emptyDir` storage volumes.

After the action finishes, you can verify that your pods were migrated from
the stopped node to the other nodes.

```console
$ kubectl get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE     IP           NODE                NOMINATED NODE   READINESS GATES
nginx-deployment-7848d4b86f-9zcl2   1/1     Running   0          3m17s   10.1.94.13   juju-37550a-k8s-3   <none>           <none>
nginx-deployment-7848d4b86f-f5ts7   1/1     Running   0          16s     10.1.94.15   juju-37550a-k8s-3   <none>           <none>
nginx-deployment-7848d4b86f-pzqkt   1/1     Running   0          3m17s   10.1.94.14   juju-37550a-k8s-3   <none>           <none>
nginx-deployment-7848d4b86f-wpsdn   1/1     Running   0          16s     10.1.94.16   juju-37550a-k8s-3   <none>           <none>
```

## Bringing Kubernetes Worker Back Online

Duration: 2:00

Once the worker is ready to be brought back into circulation, you can bring
it back online by running:

```console
juju run-action --wait kubernetes-worker/0 resume
```
[note status="Rebalancing of the cluster"]
Pods that were previously migrated from this node won't be automatically
migrated back once it's back online.
[/note]
