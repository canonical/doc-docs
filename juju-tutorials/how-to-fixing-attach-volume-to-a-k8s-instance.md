[comment]: <> (How To Perform Fixing Attach Volume to Instance)

## pre-install

Duration: 120:00

This description of how the deployment of `charmed kubernetes` is performed on a OpenStack server, where OpenStack is deployed to another OpenStack. (openstack-on-openstack).

There are two assumptions:
1. juju is configured to use the OpenStack cloud
2. openstack is configured with valid credentials (e.g. `source novarc`)

### OpenStack

Deploy the openstack-base bundle with the custom overlay.

##### Memory overlay
```yaml
applications:
  nova-compute:
    annotations:
    constraints: mem=8G
```

##### Deployment

```bash
git clone https://github.com/openstack-charmers/openstack-bundles.git
juju deploy ./openstack-bundles/stable/openstack-base/bundle.yaml --overlay ./openstack-bundles/stable/overlays/openstack-base-virt-overlay.yaml --overlay ./memory-overlay.yaml
```

##### initialize and unseal vault and create self-signed root CA

1. Vault need to be installed.
   ```bash
   snap install vault
   ```
2. Wait for the `vault/0` unit to be in the `blocked` state with message `Vault needs to be initialized`.
3. Initialize the vault deployment.
   ```bash
   export VAULT_ADDR="http://$(juju status | grep vault/0 | tr -s " " | cut -d" " -f5):8200"
   vault operator init -key-shares=5 -key-threshold=3 > vault-keys
   ```
4. Unseal all keys and generate root CA.
   ```bash
   for key in $(cat vault-keys | grep -E "Unseal Key .*: " | cut -c15-)
   do 
       vault operator unseal $key
   done
   ```
5. Authorize the vault unit.
   ```bash
   export VAULT_TOKEN=$(cat vault-keys | grep -E "Initial Root Token: " | cut -c21-)
   vault token create -ttl=10m
   juju run-action --wait vault/leader authorize-charm token=$VAULT_TOKEN
   juju run-action --wait vault/leader generate-root-ca
   ```
   
##### add external L3 connectivity

1. Get OpenStack admin net.
   ```bash
   export ADMIN_NET=$(openstack network list | grep admin_net | cut -d"|" -f3 | tr -d " ")
   ```
2. Create new port for each nova-compute unit. (by default there are 3 units)
   ```bash
   unset BRIDGE_INTERFACE_MAPPING  # clear value

   for i in 0 1 2
   do
       # create new port
       export NEW_PORT_$i_ID=$(openstack port create --network $ADMIN_NET data-port | grep ' id ' | cut -d"|" -f3 | tr -d " ")
       echo "  create new port with id $NEW_PORT_$i_ID"
    
       # get ovn chassis ip
       export OVN_CHASSIS_$i_IP=$(juju status | grep ovn-chassis/$i | tr -s " " | cut -d" " -f5)
       echo "  found ovn-chassis/$i ip address $OVN_CHASSIS_$i_IP"
    
       # get ovn chassis id
       export OVN_CHASSIS_$i_ID=$(openstack server list | grep -E $OVN_CHASSIS_$i_IP | cut -d"|" -f3 | tr -d " ")
       echo "  found ovn-chassis/$i id $OVN_CHASSIS_$i_ID"
    
       # add port to ovn chassis
       openstack server add port $OVN_CHASSIS_$i_ID $NEW_PORT_$i_ID
       echo "  add port $NEW_PORT_$i_ID to server $OVN_CHASSIS_$i_ID"
    
       # get port mac address
       export NEW_PORT_$i_MAC_ADDRESS=$(openstack port show $NEW_PORT_$i_ID | grep mac_address | cut -d"|" -f3 | tr -d " ")
       echo "  port $NEW_PORT_$i_ID mac address $NEW_PORT_$i_MAC_ADDRESS"
    
       # set bridge interface mapping variable
       export BRIDGE_INTERFACE_MAPPING="$BRIDGE_INTERFACE_MAPPING br-ex:$NEW_PORT_$i_MAC_ADDRESS"
   done
   ```
3. Configure the bridge interface mapping in the `ovn-chassis` application.
   ```bash
   juju config ovn-chassis bridge-interface-mappings="$BRIDGE_INTERFACE_MAPPING"
   ```
   
##### final configuration

The final step is to configure juju to use deployed OpenStack as cloud and configure network on OpenStack.

[note status="OpenStack credentials"]
Don't forget to get credentials for the new OpenStack. (e.g. `source openrcv3_project`)
[/note]

1. To access the OpenStack cloud is necessary to set up flavors, images, networks and security. Detailed instructions can be found on [openstack-base][OpenStackBase] in the "Accessing the cloud" section.
    [note status="OpenStack network"]
    Above Openstack, use these network settings to configure the subnet.
    ```yaml
    gateway: 10.5.0.1
    external CIDR: 10.5.0.0/16
    floating IP range: 10.5.150.0:10.5.200.254
    internal CIDR: 192.168.21.0/24
    nameserver: 10.245.168.6
    ```
    [/note]

2. The last thing is to add a new OpenStack cloud to Juju, create new credentials and create a controller. See the [OpenStack-cloud][OpenStackCloud] page for more information.
    [note status="OpenStack metadata"]
    It's necessary to create metadata for initializing a Juju cloud environment (creating the controller).
    ```bash
    juju metadata generate-image -d ~/simplestreams -i <image_id_or_name> -s <series> -r $OS_REGION_NAME -u $OS_AUTH_URL
    ```
    [/note]

### Kubernetes

Deployed charmed kubernetes on top of OpenStack with `charm-openstack-integrator` act as proxy for OpenStack and provide a control interface.

```bash
juju deploy charmed-kubernetes --overlay openstack-overlay.yaml --trust
```

Before configuration `openstack-integrator` it is necessary to source into the Openstack credential file. e.g. `source novarc`

```bash
juju config openstack-integrator auth-url=$OS_AUTH_URL region=$OS_REGION_NAME username=$OS_USERNAME user-domain-name=$OS_USER_DOMAIN_NAME password=$OS_PASSWORD project-name=$OS_PROJECT_NAME  project-domain-name=$OS_PROJECT_DOMAIN_NAME
```

### kubectl


##### install

The most common way to install `kubectl` in ubuntu is `snap`.

```bash
sudo snap install kubectl --classic
```

##### set up

To configure kubectl, you must copy the configuration file from the deployed charmed kubernetes.

```bash
mkdir ~/.kube
juju scp kubernetes-master/0:/home/ubuntu/config ~/.kube/config
```
[note status="VPN"]
If your Openstack is behind a VPN, you may need to run `sshuttle` to create a tunnel for copying from the juju unit. `sshuttle -r <user@server> <juju-network>` e.g. `sshuttle -r bastion 10.5.0.0/24` (find network with `juju status -m controller`)
[/note]

### cinder volume

##### create

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

##### attach

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
[note status="Proxy"]
A proxy must be set on OpenStack.
```bash
juju config containerd http_proxy=http://squid.internal:3128 https_proxy=http://squid.internal:3128
```
[/note]

### update QEMU

TODO

## Nova Compute - Fixing Attach Volume to Instance

Duration: 20:00

### Overview

When updating **QEMU** packages, instances that started prior to the upgrade can experience issues attaching volumes to it. 
This is due to [LP#1847361][#1847361]. In summary, this bug causes the new **QEMU** version to not recognize the old **QEMU** modules for attaching volumes to VMs. 
This bug is especially bad in Kubernetes deployments on top of Openstack. A pod will get stuck attempting to create container when it tries to attach the PVC (and by consequence, attach cinder volume to k8s worker VM). This playbook intends to explain how to identify if the **QEMU** bug is the root problem, how to fix, and proper procedure for handling the k8s situation.

### How To Identify

On the surface, attachment failures won't give much info on why it's failing. Nova log on the host will show ERROR messages similar to the following:

```bash
2020-11-04 23:48:37.871 1033447 ERROR oslo_messaging.rpc.server libvirt.libvirtError: internal error: unable to execute QEMU command 'device_add': Property 'virtio-blk-device.drive' can't find value 'drive-virtio-disk1'
```

The following logs in `/var/log/libvirt/qemu/${virsh_instance_name}.log` also indicate that it's this specific attachment issue: 

```bash
Failed to open module: /usr/lib/x86_64-linux-gnu/qemu/block-rbd.so: undefined symbol: RbdAuthMode_lookup
Failed to open module: /var/run/qemu/_Debian_1_2.11+dfsg-1ubuntu7.31_/block-rbd.so: failed to map segment from shared object
```

### Steps to Fix

There are 2 methods of fixing the issue. One method involves explicitly stopping the server and starting it back up again (reboot doesn't fix the issue). Alternatively, it's possible to migrate the instance to another host and the attach should work after migrating, as this is essentially creating a new VM on the new host. 

##### Start/Stop method

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

##### Migration Method

Simply live migrate the instance to another host. Here's an example (may need to add `--block-migration` or `--shared-migration`):

```bash
openstack server migrate --wait --live-migration $instance_uuid 
```

### K8s Attachment Issues

Extra steps must be taken when this problem manifests in a **K8s** cloud. Care must be taken to drain the **K8s** worker before actioning the instance. It may be possible to migrate the **K8s** worker instance without draining, but this hasn't been explicitly tested. These steps are taken from the [K8s Worker Maintenance Playbook][K8sWorkerMaintenance].

The steps are as follows: 

1. Drain the worker node: 

    ```bash
    juju run-action --wait kubernetes-worker/$num pause
    ```

    If the **pause** action fails due to an error with local storage, check to see if the flagged pods belong to kube-system or kube-dashboard. If so and the customer does not use this data, it's safe to run the pause command with `delete-local-storage` flag.
    
    [note status="Customer pod"]
    If it's a customer pod, be sure to ask customer how they'd like to proceed.
    ```bash
    juju run-action --wait kubernetes-worker/$num pause delete-local-storage=true
    ```
    [/note]

2. Ensure that pods look ok (likely there will be a container attempting to create as the volume attach may be broken on another worker):

    ```bash
    kubectl get pods --all-namespaces -o wide | grep -vi running
    ```
    
    Then we follow the above steps to either migrate or stop/start the instance to fix the **QEMU** attach bug. 

3. Once the instance has settled:

    ```bash
    juju run-action --wait kubernetes-worker/$num resume
    ```
    
    Repeat the above for all k8s workers to make sure that they all can attach volumes.
    

### Details

[CategoryISTeam][CategoryISTeam]

...

[#1847361]: https://bugs.launchpad.net/ubuntu/+source/qemu/+bug/1847361
[K8sWorkerMaintenance]: https://wiki.canonical.com/CDO/IS/Bootstack/Playbooks/K8sWorkerMaintenance
[CategoryISTeam]: https://wiki.canonical.com/CategoryISTeam
[OpenStackBase]: https://jaas.ai/openstack-base
[OpenStackCloud]: https://juju.is/docs/openstack-cloud
