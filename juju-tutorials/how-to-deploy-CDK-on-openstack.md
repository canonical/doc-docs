[comment]: <> (How to deploy CDK on OpenStack)

## Introduction

Duration: 20:00

This tutorial provides information on how to deploy a [CDK][charmed-distribution-kubernetes] environment on
the [OpenStack-base][openstack-base] environment.

## Requirements

There are two prerequisites you will need:

1. OpenStack is configured with valid credentials (e.g. `source novarc`, represent the OpenStack credential, belonging
to OpenStack on top of which you are deploying [OpenStack-base][openstack-base] and if you don't have it, ask your
OpenStack administrator)
1. Juju installed (see [Installing Juju][juju-install]), and the OpenStack environment is configured as cloud provider

### Kubernetes

Deploy CDK (link to CDK) on top of OpenStack using charm-openstack-integrator. The integrator will act as a proxy
between Kubernetes and OpenStack, and it will provide a control interface (e.g. creating persistent volumes).

```bash
juju deploy charmed-kubernetes --overlay openstack-overlay.yaml --trust
```

[note status="OpenStack credentials"]
In order to configure the openstack-integrator application, the OpenStack credentials need to be loaded as environment
variables. (e.g. `source novarc`, look at the "Requirements" section)
An alternative to deploying a CDK using `--trust` is to configure the `openstack-integrator` as follows:
```bash
juju config openstack-integrator auth-url=$OS_AUTH_URL region=$OS_REGION_NAME username=$OS_USERNAME user-domain-name=$OS_USER_DOMAIN_NAME password=$OS_PASSWORD project-name=$OS_PROJECT_NAME  project-domain-name=$OS_PROJECT_DOMAIN_NAME
```
[/note]

[note status="Proxy"]
If a proxy is required on the servers where you deploy it, configure the proxy in the `containered` application as well.
```bash
juju config containerd https_proxy=<proxy_ip>:<proxy_port> http_proxy=<proxy_ip>:<proxy_port>
```
[/note]

### Kubectl

#### Install

The most common way to install `kubectl` in ubuntu is as a `snap`.

```bash
sudo snap install kubectl --classic
```

#### Set up

To configure kubectl, you must copy the configuration file from the deployed charmed kubernetes. Before proceeding,
make sure that your `kubernetes-master/leader` has a floating IP address attached.

```bash
mkdir ~/.kube
juju scp kubernetes-master/0:/home/ubuntu/config ~/.kube/config
```

[note status="Tunnel"]
If your OpenStack environment is not on the local machine, or you are not working directly on the remote machine,
you may need to run it `sshuttle` to create a tunnel for copying from the juju unit.
`sshuttle -r <user@server> <juju-network>` e.g. `sshuttle -r bastion 10.5.0.0/24` (find network with
`juju status -m controller`)
[/note]

### Cinder volume

#### Create

The following yaml configuration file is used to create an 8 GB cinder volume.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-storage
spec:
  storageClassName: cdk-cinder
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
```

The next step will be to create PersistentVolumeClaim (PVC).

```bash
kubectl apply -f <pvc-configutation-file>.yaml
```

#### Attach

This step shows how to attach the PVC created in the previous step. For simplicity, we will use nginx.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-nginx
spec:
  restartPolicy: Never
  volumes:
    - name: test-volume
      persistentVolumeClaim:
        claimName: pvc-storage

  containers:
    - name: nginx-container
      image: nginx
      volumeMounts:
      - name: test-volume
        mountPath: /usr/share/nginx/html/test_volume
```

Run the command below to deploy a nginx container.

```bash
kubectl create -f <nginx_config>.yaml
```

### Update QEMU

```bash
juju run --application nova-compute 'sudo apt install --reinstall qemu'
```

...

[juju-install]: https://jaas.ai/docs/installing
[openstack-base]: https://jaas.ai/openstack-base
[charmed-distribution-kubernetes]: https://jaas.ai/canonical-kubernetes
