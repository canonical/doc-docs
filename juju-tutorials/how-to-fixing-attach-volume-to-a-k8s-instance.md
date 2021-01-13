[comment]: <> (How To Fix A Volume Attachment Failure Against An OpenStack Instance)

## Introduction

Duration: 20:00

When updating **QEMU** packages, instances that started prior to the upgrade can experience issues attaching 
volumes to it. 
This is due to [LP#1847361][#1847361]. In summary, this bug causes the new **QEMU** version to not recognize 
the old **QEMU** modules for attaching volumes to VMs. 
This bug is especially bad in Kubernetes deployments on top of OpenStack. A pod will get stuck attempting to create 
a container when it tries to attach the PVC (and by consequence, attach cinder volume to k8s worker VM). This playbook 
intends to explain how to identify if the **QEMU** bug is the root problem, how to fix, and proper procedure for 
handling the k8s situation.

### Identify attachment failures

On the surface, attachment failures won't give much info on why it's failing. Nova log on the host will show 
ERROR messages similar to the following:

```bash
2020-11-04 23:48:37.871 1033447 ERROR oslo_messaging.rpc.server libvirt.libvirtError: internal error: unable to execute QEMU command 'device_add': Property 'virtio-blk-device.drive' can't find value 'drive-virtio-disk1'
```

The following logs in `/var/log/libvirt/qemu/${virsh_instance_name}.log` also indicate that it's this specific attachment issue: 

```bash
Failed to open module: /usr/lib/x86_64-linux-gnu/qemu/block-rbd.so: undefined symbol: RbdAuthMode_lookup
Failed to open module: /var/run/qemu/_Debian_1_2.11+dfsg-1ubuntu7.31_/block-rbd.so: failed to map segment from shared object
```

## Steps to Fix

There are 2 methods of fixing the issue. One method involves explicitly stopping the server and 
starting it back up again (reboot doesn't fix the issue). Alternatively, it's possible to migrate the instance 
to another host, and the attached should work after migrating, as this is essentially creating a new VM on the new host. 

#### Start/Stop method

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

#### Migration Method

Simply live migrate the instance to another host. Here's an example 
(may need to add `--block-migration` or `--shared-migration`):

```bash
openstack server migrate --wait --live-migration $instance_uuid 
```

## K8s Attachment Issues

Extra steps must be taken when this problem manifests in a **K8s** cloud. Care must be taken to drain the **K8s** 
worker before actioning the instance. It may be possible to migrate the **K8s** worker instance without draining, 
but this hasn't been explicitly tested. These steps are taken 
from the [K8s Worker Maintenance Playbook][K8sWorkerMaintenance].

The steps are as follows: 

1. Drain the worker node: 

    ```bash
    juju run-action --wait kubernetes-worker/$num pause
    ```

    If the **pause** action fails due to an error with local storage, check to see if the flagged pods belong 
    to kube-system or kube-dashboard. If so and the customer does not use this data, it's safe to run the pause 
    command with `delete-local-storage` flag.
    
    [note status="Customer pod"]
    If it's a customer pod, be sure to ask customer how they'd like to proceed.
    ```bash
    juju run-action --wait kubernetes-worker/$num pause delete-local-storage=true
    ```
    [/note]

2. Ensure that pods look ok (likely there will be a container attempting to create as the volume attach may 
   be broken on another worker):

    ```bash
    kubectl get pods --all-namespaces -o wide | grep -vi running
    ```
    
    Then we follow the above steps to either migrate or stop/start the instance to fix the **QEMU** attach bug. 

3. Once the instance has settled:

    ```bash
    juju run-action --wait kubernetes-worker/$num resume
    ```
    
    Repeat the above for all k8s workers to make sure that they all can attach volumes.

...

[#1847361]: https://bugs.launchpad.net/ubuntu/+source/qemu/+bug/1847361
[K8sWorkerMaintenance]: https://discourse.charmhub.io/t/how-to-perform-maintenance-on-a-kubernetes-worker/3910
