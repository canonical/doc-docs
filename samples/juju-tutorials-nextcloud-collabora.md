[comment]: <> (Title goes here)
[comment]: <> (In Discourse, this will the subject of the new topic created under docs tutorials wip-docs)
[comment]: <> (check https://juju.is/tutorials/deploy-nextcloud-and-collabora-on-ubuntu)

## Introduction

[comment]: <> (Provide reading time)
Duration: H:MM

[Nextcloud](https://nextcloud.com/) provides file hosting and several collaboration tools, like contacts, calendars and task boards.

The [Collabora Online](https://nextcloud.com/collaboraonline/) plugin allows for online collaborative document editing within the Nextcloud.

They're easy to deploy and manage with [Juju](https://juju.is/). Juju makes it practical to host a personal cloud.

### Requirements

To make your way through this tutorial, you will need:

- Juju installed (see [Installing Juju](https://jaas.ai/docs/installing))
- a domain name that you control

### Testing install

Open up a terminal prompt and run the `juju version` command. If it reports a version, carry on. If not, retry the [install steps](https://jaas.ai/docs/installing).

```console
$ juju version
2.7.5
```

[note status="What to do if you get stuck"]
The best place to ask for help is the [Juju Discourse forum](https://discourse.charmhub.io/).

If you prefer chatting, visit us on [IRC](https://webchat.freenode.net/#juju) or [Matrix](https://matrix.to/#/!FTvaqcgGxWECTaZvpJ:matrix.org?via=matrix.org).  
[/note]

## Establishing a Juju controller

Duration: 5:00

We require a Juju controller. You have two options:

- Use the hosted [JAAS](https://jaas.ai) controller (recommended for using Amazon AWS, Microsoft Azure and Google Compute Platform)

- Otherwise use the bootstrap process to create a controller yourself

[note status="What is a Juju controller?"]
Juju uses a software agent called "the controller" to apply changes that you make and to monitor the system's health.  [Read more about Juju controllers](https://juju.is/docs/controllers).
[/note]

### Using the hosted JAAS controller

The Juju team provide a controller that can interact with Juju models hosted on public clouds.

```bash
$ juju login jaas
```

Using JAAS keeps costs down. You're not responsible for hosting the controller as well as your application(s).

### Bootstrap process

To bootstrap Juju onto your own hardware, we need to register your hardware first. We will collect each machine into a "cloud".

Once the cloud is available, we'll be able to create a controller into it.

#### Using your own computer hardware  

(Skip this step if you are planning to deploy to public clouds)

If you have your own server hardware, you can make use of the [manual cloud](https://jaas.ai/docs/manual-cloud) to register your computers with Juju.

The `juju add-cloud` command allows you to complete the registration process.  It takes you through an interactive prompt where you will be asked to give a name to your cloud and SSH login details (`user@10.10.10.100` in this example)

```plain
$ juju add-cloud
Cloud Types
  lxd
  maas
  manual
  openstack
  vsphere

Select cloud type: manual

Enter a name for your manual cloud: home

Enter the controller's hostname or IP address: user@10.10.10.100

Cloud "home" successfully added
You may bootstrap with 'juju bootstrap home'
```

#### Create a controller

Use the `juju bootstrap` command to create the controller.  Add the name of the cloud you wish to deploy to as an argument.

```bash
$ juju bootstrap home
```

The `juju bootstrap` command takes many options. Follow [Getting Started with Juju](https://jaas.ai/docs/getting-started-with-juju) for detailed instructions.

[note status="Which clouds does Juju support?"]
Use the `juju clouds` command to list the clouds that Juju knows about.
[/note]

## Creating a model

Duration: 0:30

A model is a workspace for our applications. It houses machines, and applications.

Complex models also include other components like networking spaces and persistent storage volumes.

```console
$ juju add-model privcloud
Added 'privcloud' model on aws/us-west-1 with credential '<credential>' for user `<user>`
```

## Adding machines

Duration: 5:00

We now need to add machines to our model. How to do this is subtly different depending on which cloud  

[details="Need to add many machines?"]
Outside of the home or small office environment, your hardware will be more complex. Multi-server we recommend installing [MAAS](https://maas.io/) and then configuring a [MAAS cloud](https://jaas.ai/docs/maas-cloud).
[/details]

We want to be conservative with our compute resources, so we'll add a large machine manually, then deploy Nextcloud and the backing database to that single machine.

[details=On a public cloud]

Juju can provision machines on our behalf:

```plain
juju add-machine --constraints="mem=8G root-disk=200G"
created machine 0
```
[/details]

[details=On a manual cloud]

We re-register the machine that's hosting the controller as a machine that's hosting the model.  

```console
$ juju add-machine sssh:user@10.10.10.100
created machine 0
```
[/details]

We'll also add a small xenial (14.04) instance to manage the `ssl-termination-proxy` charm. This will live as a container within machine 0.

```plain
$ juju add-machine lxd:0  --constraints="mem=256M"
created container 0/lxd/0
```

## Deploy Nextcloud and PostgreSQL

Duration: 2:30

[note]
If you would like a "1 click" install, you can use this command deploy everything together. By default, it will use several machines rather than making use of containers on the host:

```bash
juju deploy cs:~erik-lonroth/bundle/nextcloud-collabora-tls
```
[/note]

```bash
$ juju deploy cs:~erik-lonroth/nextcloud --to 0
Located charm "cs:~erik-lonroth/nextcloud-3".
Deploying charm "cs:~erik-lonroth/nextcloud-3".
```

Nextcloud requires a database. We'll install PostgreSQL. We allocate 120GB of storage space.

```bash
$ juju deploy postgresql --storage pgdata=120G --to 0
Located charm "cs:postgresql-203".
Deploying charm "cs:postgresql-203".
```

[note status="Multi-tenant applications"]
PostgreSQL and Nextcloud co-located on the same machine. To change this, we can add another unit of Nextcloud, then remove the `nextcloud/0` unit. Juju provides us with the ability to start small and scale up as needed.
[/note]

If you consult the `juju status` output, you'll notice an angry red "blocked" message in two places:

```plain
$ juju status
Model      Controller  Cloud/Region   Version  SLA          Timestamp
privcloud  jaas        aws/us-west-1  2.6.10    unsupported  15:31:23+13:00

App         Version  Status   Scale  Charm       Store       Rev  OS      Notes
nextcloud            blocked      1  nextcloud   jujucharms    3  ubuntu  
postgresql  10.10    active       1  postgresql  jujucharms  199  ubuntu  

Unit           Workload  Agent      Machine  Public address  Ports     Message
nextcloud/0*   blocked   idle       0        52.53.172.164   80/tcp    Need Mysql or Postgres relation to continue
postgresql/0*  active    executing  0        52.53.172.164   5432/tcp  (start) Live master (10.10)

Machine  State    DNS            Inst id              Series  AZ          Message
0        started  52.53.172.164  i-0d1e5b3a3de5eb4ca  bionic  us-west-1c  running
```

Now, we need to get the two applications to talk together.

```bash
$ juju relate nextcloud postgresql:db
```

It my take a minute or two for the system to re-configure itself, but `juju status` will soon update itself to look more like this:

```plain
$ juju status
Model      Controller  Cloud/Region   Version  SLA          Timestamp
privcloud  jaas        aws/us-west-1  2.6.10    unsupported  15:35:07+13:00

App         Version  Status  Scale  Charm       Store       Rev  OS      Notes
nextcloud            active      1  nextcloud   jujucharms    3  ubuntu  
postgresql  10.10    active      1  postgresql  jujucharms  199  ubuntu  

Unit           Workload  Agent  Machine  Public address  Ports     Message
nextcloud/0*   active    idle   0        52.53.172.164   80/tcp    Ready
postgresql/0*  active    idle   0        52.53.172.164   5432/tcp  Live master (10.10)

Machine  State    DNS             Inst id              Series  AZ          Message
0        started  52.53.172.164   i-0d1e5b3a3de5eb4ca  bionic  us-west-1c  running
```

## Accessing the Nextcloud login page

Duration: 1:30

Nextcloud is now fully operational. But accessing machine 0's public IP address (52.53.172.164) on port 80 won't yet show you a web page. We need to inform Juju to open the firewall.

```bash
juju expose nextcloud
```

If we visit the IP address now, we're presented with a login screen (and some security warnings).

![nexcloud-login|690x756](upload://1w6iUlkBJsd1HcaTt7NqsJinNe7.jpg)

What to use for the login details?

### Accessing the intial log in details for Nextcloud

Juju has created an admin account on our behalf using configuration parameters available in the charm.

We can access them via the `juju config` commands:

```plain
$ juju config nextcloud admin-username
admin
```

```bash
$ juju config nextcloud admin-password
mypassword
```

Entering these values into the form should present you with the Nextcloud application. Take the opportunity to change the admin account's password and create a non-admin user account.

![nextcloud-entry|690x756](upload://1DDWvXJrItf3vCvkEMhF0jEuBKh.png)

[note]
it is possible to set your own admin password at deployment time via the `--config` option. It wasn't done there to keep things simple.
[/note]

## Adding HTTPS

Duration: 5:00

You now need to add a DNS record to a domain for nextcloud. We'll assume `cloud.example.com`.  The A record needs to point at the IP address of the machine hosting the `ssl-termination-proxy/0` unit.

An easy way to fetch the relevant IP address is to use the filtering and formatting options of the `juju status` command:

```bash
juju status ssl-termination-proxy/0 --format=line
```
Produces
```plain
- ssl-termination-proxy/0: 54.215.139.236 (agent:idle, workload:active) 80/tcp, 443/tcp
```

With that information available, add the equivalent DNS entry:

```plain
cloud A 54.215.139.236 3600
```

Now, we set the config options for the relevant applications, we need to open the proxy to the Internet and add a relation between the ssl-termination-proxy and nextcloud-fqdn.

```bash
juju config nextcloud fqdn=cloud.example.com
juju config nextcloud-fqdn fqdns=cloud.example.com
juju expose ssl-termination-proxy
juju relate ssl-termination-proxy nextcloud-fqdn:ssl-termination
```

After a minute or so, `juju status` output will converge to something that looks like this:

```plain
$ juju status
Model      Controller  Cloud/Region   Version  SLA          Timestamp
privcloud  jaas        aws/us-west-1  2.6.8    unsupported  16:14:23+13:00

App                    Version   Status  Scale  Charm                  Store       Rev  OS      Notes
nextcloud              16.0.1.1  active      1  nextcloud              jujucharms    3  ubuntu  exposed
nextcloud-fqdn                   active      1  ssl-termination-fqdn   jujucharms    5  ubuntu  
postgresql             10.10     active      1  postgresql             jujucharms  199  ubuntu  
ssl-termination-proxy            active      1  ssl-termination-proxy  jujucharms   15  ubuntu  exposed

Unit                      Workload  Agent  Machine  Public address  Ports           Message
nextcloud/0*              active    idle   0        52.53.172.164   80/tcp          Nextcloud is OK.
postgresql/0*             active    idle   0        52.53.172.164   5432/tcp        Live master (10.10)
ssl-termination-proxy/0*  active    idle   1        54.215.139.236  80/tcp,443/tcp  Ready (cloud.example.com)
  nextcloud-fqdn/0*       active    idle            54.215.139.236                  Ready

Machine  State    DNS             Inst id              Series  AZ          Message
0        started  52.53.172.164   i-0d1e5b3a3de5eb4ca  bionic  us-west-1c  running
1        started  54.215.139.236  i-03d027ce3a8f9fc2e  xenial  us-west-1b  running
```

If yo you now access the page, you might be surprised at what you encounter:

![nextcloud-access|690x756](upload://4YGsGtYt8UF6WLZIF9tEDvdFkzZ.jpg)

Okay. The security warnings from the browser have gone, but a new one from Nextcloud has popped up. But that's okay. We can use `juju ssh` to make the recommended setting change.

### Alter Nextcloud trusted_domains setting

We now need to tweak some settings within our Nextcloud instance to allow it to understand HTTPS.

The nextcloud charm that we&#39;ve deployed supports an action that does this, `add-trusted-domain`. Include the `domain` and `index` parameters.

```plain
$ juju run-action nextcloud/0 add-trusted-domain --wait domain="localhost" index=1
Action queued with id: <id>
```

### Alter Nextcloud trusted_domains setting manually

If you want to know what's happening under the hood, you can also perform this step yourself:

#### Access remote shell securely

Juju provides a helper command that understands how to connect to units directly without needing to refer back to their IP addresses:

```bash
juju ssh nextcloud/0
```

Now, we need to find where the application is stored. We'll use the `find` command to look for the file that

```bash
sudo find / -name 'config.sample.php' 2>/dev/null
```
```plain
/var/www/nextcloud/config/config.sample.php
```

Let's look inside that directory:

```bash
ls /var/www/nextcloud/config/
```
```plain
CAN_INSTALL  config.php  config.sample.php
```

Great. `config.php` definitely looks like what we want. Let's make a backup and edit that file.

```bash
cd /var/www/nextcloud/config/
sudo cp config.php config.backup.php
```

Use your favourite editor: to edit the

```plain
sudo nano config.php
```

Edit the 'trusted_domains' line so that it includes your domain name:

```php
<?php
$CONFIG = array (
   // ...
  'trusted_domains' => array ('cloud.example.com'),
  // ...
);
```

Save the file.

### Access Nextcloud over HTTPS

Visiting your domain name with your browser should present you with a login page with no security warnings:

![nextcloud-secure|690x756](upload://57ujDAxiQtiGc5n1n7IHtW7hFnE.jpg)

## Deploying Collabora

This section will be faster, as it is mostly a duplication of the steps taken for Nextcloud.

```plain
juju deploy cs:~erik-lonroth/collabora --config nextcloud_domain=cloud.example.com
juju deploy cs:~tengu-team/ssl-termination-fqdn collabora-fqdn --config fqdns=docs.example.com
```

While that is deploying, add a DNS record for your Collabora. Here is an example:

```plain
docs A 54.215.139.236 3600
```

Now, add the relations. After they're added, your applications will automatically configure themselves.

```plain
juju relate collabora:website collabora-fqdn:website
juju relate collabora-fqdn:ssl-termination ssl-termination-proxy:ssl-termination
```

## Install the Collabora Online plugin for Nextcloud

This step requires you to adjust a setting within Nextcloud's Administration menu.

![User > Apps|690x517](upload://jTwhg7dul7tvfljW8XTvTOm3Us2.png)

![Download and enable the Collabora Online plugin|690x517](upload://t6JIPNAQgN4QcWSaFscVb2JurhH.png)

![Click the User > Settings |690x517](upload://sDtp1QLJobpsMXFBLj2XL2iZjmP.png)

![Configure the plugin with the domain name of your plugin|690x517](upload://8xDcisKhtrjZLmjKAKd65EIYCaN.png)

## Appendix: All steps as a single script

Juju is asynchronous and declarative. The commands in this tutorial can be executed in any order and will be resolved by Juju. Here they are as a single script that you can customise (be sure to update the configuration values):

```bash
juju deploy cs:~erik-lonroth/nextcloud
juju deploy cs:~erik-lonroth/collabora
juju deploy postgresql --storage pgdata=100G
juju deploy cs:~tengu-team/ssl-termination-proxy
juju deploy cs:~tengu-team/ssl-termination-fqdn collabora-fqdn
juju deploy cs:~tengu-team/ssl-termination-fqdn nextcloud-fqdn

# database configuration
juju relate postgresql:db nextcloud:postgres

# nextcloud configuration
juju relate nextcloud:website nextcloud-fqdn:website
juju relate nextcloud-fqdn:ssl-termination ssl-termination-proxy:ssl-termination
juju config nextcloud fqdn=cloud.example.com

# collabora configuration
juju relate collabora:website collabora-fqdn:website
juju relate collabora-fqdn:ssl-termination ssl-termination-proxy:ssl-termination
juju config collabora nextcloud_domain=cloud.example.com
juju config collabora-fqdn fqdns=docs.example.com

# networking
juju expose ssl-termination-proxy
```

## Acknowledgements

Juju community member @erik-lonroth built the `nextcloud` and `collabora` charms to enable this.

The Tengu team, led by @merlijn-sebrechts, provided the Let's Encrypt support.
