[comment]: <> (Title goes here)
[comment]: <> (In Discourse, this will the subject of the new topic created under "docs tutorials")
[comment]: <> ([details] are part of Discourse syntax)
[comment]: <> (Markdown cheatsheet: https://learnxinyminutes.com/docs/markdown/)

## Introduction

[comment]: <> (Provide time taken to read/test this section, in format M:SS; eg. 20 seconds)
Duration: 0:20

Share a short description (with links to the tooling) of each of the components used.

This section can also describe what you will learn at the end of the tutorial.

### Requirements

To make your way through this tutorial, you will need:

- Juju installed (see [Installing Juju](https://jaas.ai/docs/installing))
- *add here other tools*

### Testing install

Open up a terminal prompt and run the `juju version` command. If it reports a version, carry on. If not, retry the [install steps](https://jaas.ai/docs/installing).

```console
$ juju version
2.8.6-focal-amd64
```

[note status="What to do if you get stuck"]
The best place to ask for help is the [Juju Discourse forum](https://discourse.charmhub.io/).

If you prefer chatting, visit us on [IRC](https://webchat.freenode.net/#juju).
[/note]

## Establishing a Juju controller

[comment]: <> (The duration of this section; 5 minutes)
Duration: 5:00

We require a Juju controller. You have two options:

- Use the hosted [JAAS](https://jaas.ai) controller (recommended for using Amazon AWS, Microsoft Azure and Google Compute Platform)

- Otherwise use the bootstrap process to create a controller yourself

[note status="What is a Juju controller?"]
Juju uses a software agent called "the controller" to apply changes that you make and to monitor the system's health. [Read more about Juju controllers](https://juju.is/docs/controllers).
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

## Sample on sharing screenshots

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

Add your acknowledgements here.
