# Vagrant VirtualBox CoreOS with ELK Stack and Discovery using Consul and Consul-Template
This Repo contains 2 Vagrant based CoreOS machines:

1. [Etcd & Consul Server](https://github.com/stakater/coreos-vagrant-with-consul-elk/tree/master/consul-agent)
2. [Consul agent (Client machine)](https://github.com/stakater/coreos-vagrant-with-consul-elk/tree/master/etcd-consul-server)

##Etcd & Consul Server
This machine initiates etcd and also runs the consul server.

####Etcd
Etcd Discovery is handled locally through environment files, the server creates this file at `/etc/sysconfig/etcd-peers`.
This file is then fed to the etcd system unit as a drop-in in the server's cloud-config file (`etcd-consul-server/user-data.yml`).

####Consul Server
The consul server runs in a docker container from the time that the machine is started. It is submitted as a systemd Unit via the server's cloud-config file. The Unit is responsible of starting the consul server container and saving the server's ip in etcd with the key `/consul/server/ip`. So that client machines can query etcd and fetch the IP of the machine on which consul server is running.


##Consul Agent
The Consul Agent is the client machine, which runs in a cluster. You can change the number of instances in the cluster by changing the value of `num_instances` variable in the `Vagrantfile`.
The client machine starts `consul agent` and `docker registrator` via its cloud-config file (`consul-agent/user-data.yml`).

######NOTE:
The `shared` directory in both server and client machines, is mapped inside the vagrant machine to easily share files between the host and vagrant machines.

##How to Run:
1. Scroll down to [Streamlined setup](#Streamlined setup) and install all dependencies
2. Navigate to the `etcd-consul-server` directory and run `vagrant up`. Once the vagrant machine is up, you can access the Consul UI from the server's ip address and Consul port, by default : `http://172.17.8.101:8500`
3. Navigate to the `consul-agent` directory and run `vagrant up`. By default 2 vagrant client machines would start up. You would be able to see these `nodes` in the Consul UI.
4. SSH into one of the client machines, by running `vagrant ssh consul-agent-01` command. By default all client machines are named as `consul-agent-<machine-number>`
5. Run the docker-compose file for ELK as: `docker-compose -f ~/shared/elk.yaml up -d`. Once docker-compose has started all the containers successfully, you can congratulate yourself on successfully setting up  the ELK (Elasticsearch, Logstash, Kibana) Stack successfully with Consul Discovery. Now try to access Kibana at `http://172.17.9.101:5601/`
6. Now run a docker container for "log-generating" application, and map the volume which stores logs to `~/app-data/logs`
7. Run filebeat docker-compose file on the client machine which has your application running: `docker-compose -f ~/shared/filebeat.yaml up -d`. Filebeat will start sending logs to your ELK stack.

###Consul-Template
Note that both Logstash and Filebeat use `consul-tempalte` to render their templatized config files. So that Logstash can find Elasticsearch dynamically, even if Elasticserach instance or instances are changed, added, or removed. Similarly Filebeat uses templatized configuration so that it can dynamically locate Logstash nodes from consul.



###### The vagrant machines used in this set up are inspired from: https://github.com/coreos/coreos-vagrant

## Streamlined setup

1) Install dependencies

* [VirtualBox][virtualbox] 4.3.10 or greater.
* [Vagrant][vagrant] 1.6.3 or greater.

2) Clone this project and get it running!

```
git clone https://github.com/stakater/coreos-vagrant-with-consul-elk.git
cd coreos-vagrant-with-consul-elk
```

3) Startup and SSH

There are two "providers" for Vagrant with slightly different instructions.
Follow one of the following two options:

**VirtualBox Provider**

The VirtualBox provider is the default Vagrant provider. Use this if you are unsure.

```
vagrant up
vagrant ssh
```

**VMware Provider**

The VMware provider is a commercial addon from Hashicorp that offers better stability and speed.
If you use this provider follow these instructions.

VMware Fusion:
```
vagrant up --provider vmware_fusion
vagrant ssh
```

VMware Workstation:
```
vagrant up --provider vmware_workstation
vagrant ssh
```

``vagrant up`` triggers vagrant to download the CoreOS image (if necessary) and (re)launch the instance

``vagrant ssh`` connects you to the virtual machine.
Configuration is stored in the directory so you can always return to this machine by executing vagrant ssh from the directory where the Vagrantfile was located.

4) Get started [using CoreOS][using-coreos]

[virtualbox]: https://www.virtualbox.org/
[vagrant]: https://www.vagrantup.com/downloads.html
[using-coreos]: http://coreos.com/docs/using-coreos/

#### Shared Folder Setup

There is optional shared folder setup.
You can try it out by adding a section to your Vagrantfile like this.

```
config.vm.network "private_network", ip: "172.17.8.150"
config.vm.synced_folder ".", "/home/core/share", id: "core", :nfs => true,  :mount_options   => ['nolock,vers=3,udp']
```

After a 'vagrant reload' you will be prompted for your local machine password.

#### Provisioning with user-data

The Vagrantfile will provision your CoreOS VM(s) with [coreos-cloudinit][coreos-cloudinit] if a `user-data.yaml` file is found in the project directory.
coreos-cloudinit simplifies the provisioning process through the use of a script or cloud-config document.

[coreos-cloudinit]: https://github.com/coreos/coreos-cloudinit

#### Configuration

The Vagrantfile will parse a `config.rb` file containing a set of options used to configure your CoreOS cluster.
See `config.rb.sample` for more information.

## Cluster Setup

Launching a CoreOS cluster on Vagrant is as simple as configuring `$num_instances` in the `Vagrantfile` file to 3 (or more!) and running `vagrant up`.
Make sure you provide a fresh discovery URL in your `user-data` if you wish to bootstrap etcd in your cluster.

## New Box Versions

CoreOS is a rolling release distribution and versions that are out of date will automatically update.
If you want to start from the most up to date version you will need to make sure that you have the latest box file of CoreOS. You can do this by running
```
vagrant box update
```


## Docker Forwarding

By setting the `$expose_docker_tcp` configuration value you can forward a local TCP port to docker on
each CoreOS machine that you launch. The first machine will be available on the port that you specify
and each additional machine will increment the port by 1.

Follow the [Enable Remote API instructions][coreos-enabling-port-forwarding] to get the CoreOS VM setup to work with port forwarding.

[coreos-enabling-port-forwarding]: https://coreos.com/docs/launching-containers/building/customizing-docker/#enable-the-remote-api-on-a-new-socket

Then you can then use the `docker` command from your local shell by setting `DOCKER_HOST`:

    export DOCKER_HOST=tcp://localhost:2375
