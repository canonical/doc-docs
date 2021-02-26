[comment]: <> (How To Fix A Volume Attachment Failure Against An OpenStack Instance)

## Introduction

Duration: 30:00

When updating **QEMU** packages, instances that started prior to the upgrade can experience issues attaching
volumes to it.
This is due to [LP#1847361][#1847361]. In summary, this bug causes the new **QEMU** version to not recognize
the old **QEMU** modules for attaching volumes to VMs.
This bug is especially bad in Kubernetes deployments on top of OpenStack. A pod will get stuck attempting to create
a container when it tries to attach the PVC (and by consequence, attach cinder volume to k8s worker VM). This playbook
intends to explain how to identify if the **QEMU** bug is the root problem, how to fix, and proper procedure for
handling the k8s situation.

## Requirements

 1. A deployed [CDK][charmed-distribution-kubernetes] environment on top of an OpenStack environment (such as [OpenStack-base][openstack-base])
 1. OpenStack is configured with valid credentials (e.g. `source novarc`, represents
the OpenStack credentials, belonging to the managed OpenStack service on top of which the overcloud OpenStack service is deployed. If the credentials are missing, ask the managed OpenStack administrator)

See tutorial [How to deploy openstack-on-openstack, and bootstrap a Juju env on top of it][openstack-on-openstack].

## Environment preparation

For the purposes of this tutorial, it is necessary to have the PVC storage attached to any **k8s** pod and then upgrade
QEMU.

### Cinder volume

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

### QEMU update

the QEMU packages are used by the KVMs running on a `nova-compute` unit, and it could be upgraded using the following
command.

```bash
juju run --application nova-compute 'sudo apt install --upgrade qemu'
```

## Identify attachment failures

On the surface, attachment failures won't give much info on why it's failing. Nova log on the host will show
ERROR messages similar to the following:

```bash
2020-11-04 23:48:37.871 1033447 ERROR oslo_messaging.rpc.server libvirt.libvirtError: internal error: unable to execute QEMU command 'device_add': Property 'virtio-blk-device.drive' can't find value 'drive-virtio-disk1'
```

The following logs in `/var/log/libvirt/qemu/${virsh_instance_name}.log` also indicate that it's this specific
attachment issue:

```bash
Failed to open module: /usr/lib/x86_64-linux-gnu/qemu/block-rbd.so: undefined symbol: RbdAuthMode_lookup
Failed to open module: /var/run/qemu/_Debian_1_2.11+dfsg-1ubuntu7.31_/block-rbd.so: failed to map segment from shared object
```

## Steps to Fix

There are 2 methods of fixing the issue. One method involves explicitly stopping the server and
starting it back up again (reboot doesn't fix the issue). Alternatively, it's possible to migrate the instance
to another host, and the attached should work after migrating, as this is essentially creating a new VM on the new host.

### Start/Stop method

Stop the server:

```bash
openstack server stop $instance_uuid
```

Wait for the server to shutdown completely:

```bash
watch openstack server show $instance_uuid -c OS-EXT-STS:power_state -c OS-EXT-STS:task_state -c OS-EXT-STS:vm_state
```

Start the server:

```bash
openstack server start $instance_uuid
```

### Migration Method

Simply live migrate the instance to another host. Here's an example
(may need to add `--block-migration` or `--shared-migration`):

```bash
openstack server migrate --wait --live-migration $instance_uuid
```

## K8s Attachment Issues

Extra steps must be taken when this problem manifests in a **K8s** cloud. Care must be taken to drain the **K8s**
worker before actioning the instance. It may be possible to migrate the **K8s** worker instance without draining,
but this hasn't been explicitly tested. These steps on how to drain (pause and resume) the **K8s** worker are
described in [K8s Worker Maintenance Tutorial][K8sWorkerMaintenance].

[note status="Customer pod"]
If it's a custom pod, be aware that using the `delete-local-storage=true` parameter will result in the loss of locally
stored data in the pod.
[/note]

[#1847361]: https://bugs.launchpad.net/ubuntu/+source/qemu/+bug/1847361
[openstack-base]: https://jaas.ai/openstack-base
[charmed-distribution-kubernetes]: https://jaas.ai/canonical-kubernetes
[K8sWorkerMaintenance]: https://discourse.charmhub.io/t/how-to-perform-maintenance-on-a-kubernetes-worker/3910
[openstack-on-openstack]: https://discourse.charmhub.io/t/how-to-deploy-openstack-on-openstack-and-bootstrap-a-juju-env-on-top-of-it/4189
