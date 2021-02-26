[comment]: <> (How to deploy openstack-on-openstack, and bootstrap a Juju env on top of it)

## Introduction

Duration: 80:00

This tutorial has been tested on a [CDK][charmed-distribution-kubernetes] environment deployed on
the [OpenStack-base][openstack-base] environment. The OpenStack environment was deployed on top of another managed
OpenStack service.

## Requirements

There are three prerequisites needed:

1. Juju is installed (see [Installing Juju][juju-install])
1. Juju controller is up and running (see [Creating a controller][juju-controller])
1. OpenStack is configured with valid credentials (e.g. `source novarc`, represents the OpenStack credentials,
   belonging to the managed OpenStack environment on top of which [OpenStack-base][openstack-base] is deployed.
   If the credentials are missing, ask the managed OpenStack administrator)

## OpenStack

Deploy the [OpenStack-base][openstack-base] bundle with the custom overlay.

### Resources overlay

The following overlay must be used if [CDK][charmed-distribution-kubernetes] is deployed over OpenStack,
otherwise the minimum requirements would not be met, and the `kubernetes-master` (and `worker`) units
could not be deployed.

```yaml
applications:
  ceph-osd:
    storage:
      osd-devices: 40GB
  nova-compute:
    annotations:
    constraints: cores=6 mem=8G
```

### Deployment

```bash
git clone https://github.com/openstack-charmers/openstack-bundles.git
juju deploy ./openstack-bundles/stable/openstack-base/bundle.yaml --overlay ./openstack-bundles/stable/overlays/openstack-base-virt-overlay.yaml --overlay ./resources-overlay.yaml
```

### Initialize and unseal Vault and create a self-signed root CA

1. Wait for the `vault/0` unit to be in the `blocked` state with message `Vault needs to be initialized`.
   ```bash
   juju ssh vault/0
   ```
1. Initialize Vault deployment. (for more information, look at the [official documentation][vault-init])
   ```bash
   export VAULT_ADDR="http://127.0.0.1:8200"
   vault operator init -key-shares=3 -key-threshold=3 > vault-keys
   ```
1. Unseal all keys and generate root CA.
   ```bash
   for key in $(grep -E "Unseal Key .*: " vault-keys | cut -c15- | tail -3)
   do
       vault operator unseal $key
   done
   ```
1. Create a temporary token which will expire after 10 minutes. (for more information visit [page][ttl-create])
   ```bash
   export VAULT_TOKEN=$(grep -E "Initial Root Token: " vault-keys | cut -c21-)
   vault token create -ttl=10m
   ```
1. Detach from the `vault/0` unit.
1. Authorize the vault unit.
   ```bash
   export VAULT_TOKEN=$(juju ssh vault/0 'grep -E "Initial Root Token: " vault-keys | cut -c21-')
   juju run-action --wait vault/leader authorize-charm token=$VAULT_TOKEN
   juju run-action --wait vault/leader generate-root-ca
   ```

### Add external L3 connectivity

This section deals with the prerequisites for network configuration and is required for the full functionality of
the deployed OpenStack. It provides network traffic between individual units.

[note status="OpenStack credentials"]
Don't forget to load the managed OpenStack service credentials. (e.g. `source novarc`, see previous "Requirements" section), which will allow the configuration of objects for the overcloud OpenStack-base service.
[/note]

1. Get OpenStack admin net.
   ```bash
   export ADMIN_NET=$(openstack network list --format value | grep admin_net | cut -d" " -f1)
   ```
1. Create a new port for each nova-compute unit. (by default there are 3 units)
   ```bash
   unset BRIDGE_INTERFACE_MAPPING  # clear value

   for i in 0 1 2
   do
       # create new port
       eval $(openstack port create --format shell --prefix port_ --network $ADMIN_NET data-port)
       echo "  create new port with id $port_id and MAC address $port_mac_address"

       # get ovn chassis ip
       export ovn_chassis_ip=$(juju status | grep ovn-chassis/$i | tr -s " " | cut -d" " -f5)
       echo "  found ovn-chassis/$i ip address $ovn_chassis_ip"

       # get ovn chassis id
       export ovn_chassis_id=$(openstack server list -f value | awk -v IP=$ovn_chassis_ip '$0 ~ IP {print $1}')
       echo "  found ovn-chassis/$i id $ovn_chassis_id"

       # add port to ovn chassis
       openstack server add port $ovn_chassis_id $port_id
       echo "  add port $port_id to server $ovn_chassis_id"

       # set bridge interface mapping variable
       export BRIDGE_INTERFACE_MAPPING="$BRIDGE_INTERFACE_MAPPING br-ex:$port_mac_address"
   done
   ```
1. Configure the bridge interface mapping in the `ovn-chassis` application.
   ```bash
   juju config ovn-chassis bridge-interface-mappings="$BRIDGE_INTERFACE_MAPPING"
   ```

[note status="configuring a new interface for ovn-chassis"]
After adding and configuring new ports to the `ovn-chassis` units, you need to run the
`juju resolve ovn-chassis/<unit_number>` twice for each unit.
[/note]

### Final configuration

The final step is to configure Juju to use the deployed OpenStack environment as cloud provider, and configure it

[note status="OpenStack credentials"]
Don't forget to get credentials for the new OpenStack. (e.g. `source openrc`, which should be found in
[OpenStack bundles repository][openstack-bundles-base])
[/note]

1. In order to use the overcloud OpenStack service, several configurations are required: flavors, images, networks and
   security groups. Detailed instructions can be found on [OpenStack-base][openstack-base] in the "Accessing the cloud"
   section.
   1. Images - creating focal image
      ```bash
      curl https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img | openstack image create --public --container-format=bare --disk-format=qcow2 focal
      ```
   1. Flavors - creating four basic types of flavors and one use for the bootstrap controller
      ```bash
      openstack flavor create --vcpus 1 --ram 1024 --disk 10 --ephemeral 0 m1.tiny
      openstack flavor create --vcpus 2 --ram 2048 --disk 20 --ephemeral 0 m1.small
      openstack flavor create --vcpus 2 --ram 4096 --disk 20 --ephemeral 0 m1.medium
      openstack flavor create --vcpus 4 --ram 4096 --disk 20 --ephemeral 0 m1.large
      openstack flavor create --vcpus 2 --ram 3584 --disk 20 --ephemeral 0 m1.controller
      ```
   1. Networks - two networks (internal and external) are required for openstack to work
      ```bash
      openstack network create --external --provider-network-type flat --provider-physical-network physnet1 external_net
      openstack subnet create --subnet-range $EXT_SUBNET_RANGE --no-dhcp --gateway $EXT_GATEWAY --network external_net --allocation-pool start=$EXT_POOL_START,end=$EXT_POOL_END external_subnet
      openstack network create internal_net
      openstack subnet create --network internal_net --subnet-range $INT_SUBNET_RANGE --dns-nameserver $INT_DNS internal_subnet
      openstack router create provider-router
      openstack router set --external-gateway external_net provider-router
      openstack router add subnet provider-router internal_subnet
      ```
      [note status="configuration example"]
      ```bash
      export EXT_SUBNET_RANGE=10.5.0.0/16
      export EXT_GATEWAY=10.5.0.1
      export EXT_POOL_START=10.5.150.0
      export EXT_POOL_END=10.5.200.254
      export INT_SUBNET_RANGE=192.168.21.0/24
      export INT_DNS=10.5.0.2
      ```
      [/note]
   1. Security - needed to enable ssh connectivity
      ```bash
      PROJECT_ID=$(openstack project list -f value -c ID --domain admin_domain)
      SECGRP_ID=$(openstack security group list --project $PROJECT_ID | awk '/default/{print$2}')

      openstack security group rule create $SECGRP_ID --protocol icmp --ingress --ethertype IPv4
      openstack security group rule create $SECGRP_ID --protocol icmp --ingress --ethertype IPv6
      openstack security group rule create $SECGRP_ID --protocol tcp --ingress --ethertype IPv4 --dst-port 22
      openstack security group rule create $SECGRP_ID --protocol tcp --ingress --ethertype IPv6 --dst-port 22
      ```
   1. Quota - change quotas to increase the number of instances and security groups that can be created
      ```bash
      PROJECT_ID=$(openstack project list -f value -c ID --domain admin_domain)
      openstack quota set --instances 40 --secgroups 40 $PROJECT_ID
      ```
   1. Keypair - allow ssh access to instances using a customer ssh public key
      ```bash
      openstack keypair create --public-key <path_to_publick_key> mykey
      ```

1. The last thing is to add a new OpenStack cloud to Juju, create new credentials and create a controller.
   See [Using OpenStack with Juju][openstack-cloud] for more information.

   [note status="Tunnel"]
   If your OpenStack environment is not on the local machine, or the remote machine is not reachable directly,
   you may need to run `sshuttle` to redirect all the traffic via a ssh tunnel.
   `sshuttle -r <user@server> <juju-network>` e.g. `sshuttle -r jumphost 10.5.0.0/16` (find network with
   `juju status -m controller`)
   [/note]

   1. Add OpenStack as a Juju cloud provider.
      ```bash
      juju add-cloud
      ```
      ```yaml
      # cloud configuration example
      clouds:
        <cloud_name>:
          type: openstack
          auth-types: [userpass]
          endpoint: https://<keystone_internal_ip>:5000/v3
          regions:
            RegionOne:
              endpoint: https://<keystone_internal_ip>:5000/v3
          ca-certificates:
          - |
            -----BEGIN CERTIFICATE-----
            <your_certificate>
            -----END CERTIFICATE-----
      ```
   1. Add credentials to the "openstack" cloud.
      ```bash
      juju add-credential <cloud_name>
      ```
      ```yaml
      # credentials example
      credentials:
        <cloud_name>:
          <credentials name>:
            auth-type: userpass
            username: admin
            password: <user_password>
            domain-name: null
            project-domain-name: admin_domain
            tenant-name: admin
            user-domain-name: admin_domain
            version: "3"
      ```
   1. Allow multiple units to land on 3 nova-compute units (edit disk, ram and cpu ratio).
      ```bash
      juju config nova-cloud-controller disk-allocation-ratio=10 ram-allocation-ratio=1 cpu-allocation-ratio=4
      ```
   1. It's necessary to create metadata for initializing a Juju cloud environment (creating the controller).
      ```bash
      IMG_ID=$(openstack image list --name focal -c ID -f value)
      juju metadata generate-image -d ./simplestreams -i $IMG_ID -s focal -r $OS_REGION_NAME -u $OS_AUTH_URL
      ```
   1. Bootstrap the controller.
      ```bash
      EXT_NET=$(openstack network list --name external_net -c ID -f value)
      INT_NET=$(openstack network list --name internal_net -c ID -f value)
      juju bootstrap --metadata-source ./simplestreams --model-default external-network=$EXT_NET --model-default network=$INT_NET --model-default use-floating-ip=true --model-default use-default-secgroup=true --config default-series=focal <cloud_name> <controller_name>
      ```
      [note status="Proxy"]
      If the environment used to deploy the instances requires a proxy, the following must also be used:
      `--model-default http-proxy=<proxy_ip>:<proxy_port> --model-default https-proxy=<proxy_ip>:<proxy_port>`.
      [/note]

...

[juju-install]: https://jaas.ai/docs/installing
[juju-controller]: https://juju.is/docs/creating-a-controller
[openstack-base]: https://jaas.ai/openstack-base
[openstack-cloud]: https://juju.is/docs/openstack-cloud
[openstack-bundles-base]: https://github.com/openstack-charmers/openstack-bundles/tree/master/stable/openstack-base
[charmed-distribution-kubernetes]: https://jaas.ai/canonical-kubernetes
[vault-init]: https://www.vaultproject.io/docs/commands/operator/init
[ttl-create]: https://www.vaultproject.io/docs/commands/token/create
