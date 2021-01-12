[comment]: <> (Juju Tricks)

## Introduction

Duration: 0:01

This article is compilation of CLI tricks, useful for recovery of Juju deployments after services or hosts reboot, crash, or network components become unreachable.

### Requirements

There is no specific setup precedure for this tutorial because it contains commands that are applicable in various situations in any Juju deployment. Most of the commands are using just standard CLI utilities like `grep`, `sed` or `awk`.

[note status="What to do if you get stuck"]
The best place to ask for help is the [Charmhub Discourse forum](https://discourse.charmhub.io/).

If you prefer chatting, visit us on [IRC](https://webchat.freenode.net/#juju).
[/note]

## Tricks on broken units, machines or models

Duration: N/A

[note status="Direct reachability"]
For the commands in this section to work, the targeted units or machines must be directly reachable via `juju ssh`. If, for example, LXDs are using a network that is reachable only from the LXD host machine, you will encounter connectivity errors.
[/note]

### Restarting units and machines based on their state

To restart units that are in a particular state (e.g. `lost` or `down`) use the following command and adjust the first part, `state=<TARGET_STATE>`, to match the state of the units that you want to restart. The command, as it is, will restart the units in state `down`.

```console
state=down; juju status --format oneline | grep "agent:${state}" | egrep -o '[a-z0-9-]+/[0-9]+' | sort -u | xargs -P4 -I@ bash -c 'unit=@; juju ssh $unit "sudo service jujud-unit-${unit/\//-} restart"'
```

To restart machines in a particular state (including LXDs) use the following command and adjust the first part, `state=<TARGET_STATE>`, to match the state of the machines that you want to restart. The command, as it is, will restart the machines in state `down`.

```console
state=down; juju machines | sed "1 d" | awk -v state=${state} '{if ($2 == state) print $1}' | xargs -P4 -I@ bash -c 'machine=@; svc=${machine//\//-}; juju ssh ${machine%:} "sudo service jujud-machine-${svc}" restart'
```

### Restarting all units and machines

To restart all units, regardless of their state, run the following command.

```console
juju status --format oneline | egrep -o '[a-z0-9-]+/[0-9]+' | sort -u | xargs -P4 -I@ bash -c 'unit=@; juju ssh $unit "sudo service jujud-unit-${unit/\//-} restart"
```

To restart all machines, regardless of their state, run the following command.

```console
juju machines | sed "1 d" | awk '{print $1}' | xargs -P4 -I@ bash -c 'machine=@; svc=${machine//\//-}; juju ssh ${machine%:} "sudo service jujud-machine-${svc}" restart'
```

### Retrying failed hooks

If there are multiple units with failed hooks, you can retry to re-run them with `juju resolved --all`

## Juju relations

Duration: N/A

### Inspecting relation data

You can use `juju show-unit` command to see details about the unit's relations, including relation data. For example, to inspect relation `cni` of the unit `kubernetes-master/0`.

```console
juju show-unit kubernetes-master/0 --endpoint cni
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

### Running `juju debug-hooks` on Juju backed by Kubernetes

Current stable Juju release (2.8) does not support running `juju debug-hooks` command on model/units backed by Kubernetes. This feature will become available in 2.9 release (see [Release Notes](https://discourse.charmhub.io/t/juju-2-9-beta-1-release-notes/3642)). You can, however, achieve similar debugging capabilities using the following trick.

Let's say we have a `gunicorn` unit with `db-relation-joined` hook failing and we want to find out what's causing the failure.

[note status="Placeholders"]
In the following commands, we will use `<JUJU_MODEL>` and `<JUJU_UNIT>` as placeholders for actual unit and model names, replace them with approprate values.
[/note]

Get a shell on the operator pod using `microk8s.kubectl`:

```console
microk8s.kubectl  exec -it -n <JUJU_MODEL> pod/<JUJU_UNIT>-operator-0 bash
```

Replace faulty hook with a placeholder script that will dump hook's environment variables. Note the `sleep` in this script, it will give us one hour to do our debugging. If you suspect you'll need longer, put in appropriate value. Don't be afraid to go big, we will kill the sleep when we are done debugging.

```console
cd agents/unit-<JUJU_UNIT>/charm/hooks
mv db-relation-joined db-relation-joined.old
cat > db-relation-joined <<EOF
#!/bin/bash
env
sleep 3600
exit 1
EOF
chmod 755 db-relation-joined
```

Log out of the operator pod and run `juju resolved`, this will re-run failed hook that is now replaced with our placeholder script and we will see all the dumped environment variables in the `juju debug-log`

```console
juju resolved <JUJU_UNIT>
juju debug-log
```

Example of the output in `juju debug-log` is something like this:

```console
application-gunicorn: 13:30:16 DEBUG unit.gunicorn/19.db-relation-joined JUJU_UNIT_NAME=gunicorn/19
application-gunicorn: 13:30:16 DEBUG unit.gunicorn/19.db-relation-joined JUJU_AGENT_SOCKET_ADDRESS=@/var/lib/juju/agents/unit-gunicorn-19/agent.socket
```

Copy every line that's produced from `db-relation-joined` hook. Log back into the unit and export the dumped variables. This will simulate environment in which the juju hooks are typicaly running with all the necessary variables set.

```console
microk8s.kubectl  exec -it -n <JUJU_MODEL> pod/<JUJU_UNIT>-operator-0 bash
for i in $(cat|awk '{print $5}'); do export $i; done
[paste previously copied lines from the debug-log]
^d
cd $PWD
```

Now recreate a proper hook file and run it to debug it.

```console
mkdir hooks/tmp/
ln -s ../../src/charm.py hooks/tmp/$JUJU_HOOK_NAME
hooks/tmp/$JUJU_HOOK_NAME
```

Output of the hook can look like this:

```console
Traceback (most recent call last):
  File "lib/ops/model.py", line 697, in _run
    result = run(args, check=True, **kwargs)
  File "/usr/lib/python3.6/subprocess.py", line 438, in run
    output=stdout, stderr=stderr)
subprocess.CalledProcessError: Command '('relation-set', '-r', '1', 'database=mydbname', '--app=True')' returned non-zero exit status 1.
```

When you are done with debugging, kill the sleep from our placeholder script.

```console
pkill -x sleep
```

### Inspecting state of reactive charms

It's real simple to check states of the reactive charm. States (or flags) are used by the reactive framework to transition the components of a unit (relations, configurations, ...) from a not-ready to a ready status. Inspecting them can help you troubleshoot a possible bug in the charm. In the example below we will inspect states of the unit `flannel/1`

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
Changing values in the Juju database can corrupt your Juju installation. Proceed with caution and doublecheck before making any changes.
[/note]

Sometimes it may be necessary to access Juju's database directly, although it should be considered as last resort solution. To log into the Juju's database, you must first log into the Juju controller machine.

```console
juju ssh -m controller 0
```

Once there, you can log into the Juju's mongo database.

```console
mongo --sslAllowInvalidCertificates --authenticationDatabase admin --ssl -u $(sudo awk '/tag/ {print $2}' /var/lib/juju/agents/machine-?/agent.conf) -p $(sudo awk '/statepassword/ {print $2}' /var/lib/juju/agents/machine-?/agent.conf) localhost:37017/juju
```

### Manual change of Openstack endpoint certificates

In case you are running a Juju controller on top of Openstack and for some reason the CA certificates on the endpoints change, you can't access the endpoint anymore with "certificate signed by unknown authority", the only way to fix it is to update Juju's mongodb directly.

Start by logging into the controller node and then into the mongo interactive shell. Once you are in the mongo shell, update the certificate field for your cloud in the `clouds` collection.

```console
juju:PRIMARY> db.clouds.update({"_id": "openstack_cloud"}, {$set: {"ca-certificates": [""]}})
```

Certificate must be PEM-formatted one-liner with newlines replaced by `\n`.

`-----BEGIN CERTIFICATE-----\n<CERTIFICATE_CONTENT>\n\n-----END CERTIFICATE-----\n`

### Debugging stuck transactions

If a transaction is stuck and you want to figure out what's going on, look at the 'cleanups' collection.

```console
juju:PRIMARY> db.cleanups.find().pretty()
```

### Dumping database content

Although `juju create-backup` exists, you can dump raw Juju database by logging into the controller and running the following:

```console
datestamp=`date +"%Y-%m-%d_%H-%M-%S"`
conf=/var/lib/juju/agents/machine-*/agent.conf
user=`sudo grep tag $conf | cut -d' ' -f2`
password=`sudo grep statepassword $conf | cut -d' ' -f2`
mongodump -h 127.0.0.1:37017 --username $user --password $password --authenticationDatabase admin  --db juju -o mongodump --ssl --sslAllowInvalidCertificates
tar -zcf "juju-mongodump-${datestamp}.tar.gz" mongodump
```
