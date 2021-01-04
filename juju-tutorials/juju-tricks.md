[comment]: <> (Juju Tricks)

## Introduction

Duration: 0:01

This article is compilation of CLI tricks, useful for recovery of juju deployments after OS services' hosts reboot / crash / network splits / etc.

### Requirements

There is no specific setup precedure for this tutorial because it contains commands that are applicable in various situations in any juju deployment. Most of the commands are using just standard cli utilities like `grep`, `sed` or `awk`.

[note status="What to do if you get stuck"]
The best place to ask for help is the [Charmhub Discourse forum](https://discourse.charmhub.io/).

If you prefer chatting, visit us on [IRC](https://webchat.freenode.net/#juju).
[/note]

## Tricks on broken units / machines / models

Duration: N/A

[note status="Direct reachability"]
For the commands in this section to work, the targeted units or machines must be directly reachable via `juju ssh`. If, for example, LXCs are using network that is reachable only from the LXC host machine, you will encounter connectivity errors.
[/note]

### Restarting units and machines based on their state

To restart units that are in particular state (e.g. `lost` or `down`) use following command and adjust the first part, `state=<TARGET_STATE>`, to match state of the units that you want to restart. The command, as it is, will restart units in state `down`.

```console
state=down; juju status --format oneline | grep "agent:${state}" | xargs -l4 | column -t | egrep -o '[a-z0-9-]+/[0-9]+' | sort -u | xargs -P4 -I@ bash -c 'unit=@; juju ssh $unit "sudo service jujud-unit-${unit/\//-} restart"'
```

To restart machine in particular state (including LXCs) use the following command and adjust the first part, `state=<TARGET_STATE>`, to match state of the machines that you want to restart. The command, as it is, will restart machines in state `down`.

```console
state=down; juju machines | grep -v 'Machine' | awk -v state=${state} '{if ($2 == state) print $1}' | xargs -P4 -I@ bash -c 'machine=@; svc=${machine//\//-}; svc=${svc%:}; juju ssh ${machine%:} "sudo service jujud-machine-${svc}" restart'
```

### Restarting all units and machines

To restart all units, regardless of their state, run following command.

```console
juju status --format oneline | xargs -l4 | column -t | egrep -o '[a-z0-9-]+/[0-9]+' | sort -u | xargs -P4 -I@ bash -c 'unit=@; juju ssh $unit "sudo service jujud-unit-${unit/\//-} restart"
```

To restart all machines, regardless of their state, run following command.

```console
juju machines | grep -v 'Machine' | awk '{print $1}' | xargs -P4 -I@ bash -c 'machine=@; svc=${machine//\//-}; svc=${svc%:}; juju ssh ${machine%:} "sudo service jujud-machine-${svc}" restart'
```

## Tricks on debugging unit relations

Duration: N/A

### Inspecting relation data

To see data of the specific relation on certain unit, run the following command and adjust variables `unit` and `relation`  to match unit and relation that you wish to inspect. This example command will print data from the relation `cni` of the unit `kubernetes-master/0`.

```console
unit=kubernetes-master/0;relation=cni; juju run --unit $unit "relation-ids $relation | xargs -I_@ sh -c 'relation-list -r _@ | xargs -I_U sh -c \"relation-get -r _@ - _U | sed s,^,_U:, 2>&1\"'"
```

### Manually setting relation data

To manually set data on some relation, we must first find its relation ID. Lets say we want to change value of the `cni-conf-file` on the `cni` relation between units `kubernetes-worker/0` and `flannel/1`. We run this command to determine relation ID.

```console
juju run --unit kubernetes-master/0 'relation-ids cni'
cni:11
```

Then we can set the new value by running

```console
juju run --unit flannel/1 "relation-set -r cni:11 cni-conf-file=10-flannel.conflist"
```

### Triggering relation-changed hook

The easiest way to trigger `relation-changed` hook is to change data in the relation using dummy variable that will be ignored by the other end of the relation. See chapter above, "Manually setting relation data" to see how to run `relation-set`.

[note status="Reactive charms"]
For the reactive charms, use 'juju run --unit foo/0 charms.reactive' to set/unset the flags and wait for the update-status hook to run
[/note]

## Debugging

Duration: N/A

## Inspecting state of reactive charms

It's real simple to check states of the reactive charm. In the example below we will inspect states of the unit `flannel/1`

```console
juju run --unit flannel/1 'charms.reactive -p get_states'
{'cni.configured': None,
 'cni.connected': None,
 'cni.is-master': None,
 'endpoint.cni.changed.cni-conf-file': None,
 'endpoint.cni.changed.egress-subnets': None,
 'endpoint.cni.changed.ingress-address': None,
 'endpoint.cni.changed.is_master': None,
 'endpoint.cni.changed.private-address': None,
 'endpoint.cni.joined': None,
 'etcd.available': {'conversations': ['reactive.conversations.etcd.global'],
                    'relation': 'etcd'},
 'etcd.connected': {'conversations': ['reactive.conversations.etcd.global'],
                    'relation': 'etcd'},
 'etcd.tls.available': {'conversations': ['reactive.conversations.etcd.global'],
                        'relation': 'etcd'},
 'flannel.binaries.installed': None,
 'flannel.cni.available': None,
 'flannel.etcd.credentials.installed': None,
 'flannel.network.configured': None,
 'flannel.service.installed': None,
 'flannel.service.started': None,
 'flannel.version.set': None}
```

## Accessing juju internal database

Duration: N/A

[note status="Caution"]
Changing values in the juju database can corrupt your juju installation. Proceed with caution and doublecheck before making any changes.
[/note]

Sometimes it may be necessary to access juju's database directly, although it should be considered as last resort solution. To log into the juju's database, you must first log into the juju controller machine.

```console
juju ssh -m controller 0
```

Once there, you can log into the juju's mongo database.

```console
mongo --sslAllowInvalidCertificates --authenticationDatabase admin --ssl -u $(sudo awk '/tag/ {print $2}' /var/lib/juju/agents/machine-?/agent.conf) -p $(sudo awk '/statepassword/ {print $2}' /var/lib/juju/agents/machine-?/agent.conf) localhost:37017/juju
```

### Manual change of Openstack endpoint certificates

In case you are running juju controller on top of Openstack and for some reason the CA certificates on the endpoints change, you can't access the endpoint anymore with "certificate signed by unknown authority", the only way to fix it is to update the juju mongodb directly.

Start by logging into the controller node and then into the mongo interactive shell. Once you are in the mongo shell, update certificate field for your cloud in `clouds` collection.

``console
juju:PRIMARY> db.clouds.update({"_id": "openstack_cloud"}, {$set: {"ca-certificates": [""]}})
```

Certificate must be PEM-formatted one-liner with newlines replaced by `\n`.

```
-----BEGIN CERTIFICATE-----\n<CERTIFICATE_CONTENT>\n\n-----END CERTIFICATE-----\n
```

### Debugging stuck transactions

If a transaction is stuck and you want to figure out what's going on, look at the 'cleanups' collection.

```console
juju:PRIMARY> db.cleanups.find().pretty()
```

### Dumping database content

There already is `juju create-backup` tool, but in case it's not working or you just want to dump raw juju  database, you can log into the controller and run following.

```console
datestamp=`date +"%Y-%m-%d_%H-%M-%S"`
conf=/var/lib/juju/agents/machine-*/agent.conf
user=`sudo grep tag $conf | cut -d' ' -f2`
password=`sudo grep statepassword $conf | cut -d' ' -f2`
mongodump -h 127.0.0.1:37017 --username $user --password $password --authenticationDatabase admin  --db juju -o mongodump --ssl --sslAllowInvalidCertificates
tar -zcf "juju-mongodump-${datestamp}.tar.gz" mongodump
```