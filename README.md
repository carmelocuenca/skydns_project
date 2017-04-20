# CoreOS Vagrant and SkyDNS

## What is this code?

This code defines a CoreOS cluster infraestructure with 3 nodes.
A random node runs a fleet unit which mainly setups a postgresql service, the
unit also registers the container ip (a flannel ip) with  a name (*db*) in the
cluster etcd service to being used by a Sky DNS service.

In order to check the registration, all nodes run two fleet units, one is a
skydns container service to provide DNS service to the second  unit. The second
container uses the skydns container like a DNS and pings by name to *db*.

Something like this

```
$ fleetctl list-units
UNIT			MACHINE				ACTIVE	SUB
ping.service		43130b05.../172.17.8.103	active	running
ping.service		458749bd.../172.17.8.101	active	running
ping.service		c1449516.../172.17.8.102	active	running
postgresql.service	43130b05.../172.17.8.103	active	running
skydns.service		43130b05.../172.17.8.103	active	running
skydns.service		458749bd.../172.17.8.101	active	running
skydns.service		c1449516.../172.17.8.102	active	running
```

## Registration
```
ExecStart=/usr/bin/docker run --rm --name some-postgres \
  -e "POSTGRES_USER=${POSTGRES_USER}" -e "POSTGRES_PASSWORD=${POSTGRES_PASSWORD}" \
  postgres
ExecStartPost=/bin/bash -c "sleep 10"
ExecStartPost=/bin/bash -c '\
/usr/bin/etcdctl set /skydns/local/example/db \
"{ \\"host\\":  \\"$(/usr/bin/docker inspect --format "{{ .NetworkSettings.IPAddress }}" some-postgres)\\" }"'
```

## Discovery
The dns server
```
ExecStartPre=/usr/bin/etcdctl set /skydns/config \
 '{"dns_addr":"0.0.0.0:53", "domain": "example.local.", "ttl":30}'
ExecStart=/usr/bin/docker run --rm --name some-dns \
  -e ETCD_MACHINES="http://$public_ipv4:2379" \
  skynetservices/skydns
```

The dns client
```
ExecStart=/bin/bash -c '\
  /usr/bin/docker run --rm --name some-ping \
  --dns $(/usr/bin/docker inspect --format "{{ .NetworkSettings.IPAddress }}" some-dns) \
  busybox /bin/sh -c "while true; do /bin/ping -c 1 db.example.local; sleep 10; done"'
````

## Checking
- Checking *etcd2*
```
$ for i in `seq 1 3`; do vagrant ssh core-0$i -c 'etcdctl cluster-health'; done
...
cluster is healthy
```
- Checking *fleet*
```
$ for i in `seq 1 3`; do vagrant ssh core-0$i -c 'fleetctl list-machines'; done
```
...
MACHINE		IP		METADATA
1b7f7149...	172.17.8.103	-
9ca46cdb...	172.17.8.101	-
9fed7a0f...	172.17.8.102	-

- Checking *postgresql.service*
```
 $ for i in `seq 1 3`; do vagrant ssh core-0$i -c 'fleetctl list-units | grep postgresql.service'; done
 ...
 postgresql.service	70915824.../172.17.8.101	active	running
```
- Checking *registrator*
```
 $ for i in `seq 1 3`; do vagrant ssh core-0$i -c '/usr/bin/etcdctl get /skydns/local/example/db'; done
 ...
 postgresql.service	70915824.../172.17.8.101	active	running
```

- Checking all
After a while...
```
 $ for i in `seq 1 3`; do vagrant ssh core-0$i -c '/usr/bin/journalctl -u ping.service'; done
 ...
 Apr 20 19:47:04 core-01 bash[2297]: 64 bytes from 10.1.93.3: seq=0 ttl=60 time=1
```

# CoreOS Vagrant

This repo provides a template Vagrantfile to create a CoreOS virtual machine using the VirtualBox software hypervisor.
After setup is complete you will have a single CoreOS virtual machine running on your local machine.

## Contact
IRC: #coreos on freenode.org

Mailing list: [coreos-dev](https://groups.google.com/forum/#!forum/coreos-dev)

## Streamlined setup

1) Install dependencies

* [VirtualBox][virtualbox] 4.3.10 or greater.
* [Vagrant][vagrant] 1.6.3 or greater.

2) Clone this project and get it running!

```
git clone https://github.com/coreos/coreos-vagrant/
cd coreos-vagrant
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

The Vagrantfile will provision your CoreOS VM(s) with [coreos-cloudinit][coreos-cloudinit] if a `user-data` file is found in the project directory.
coreos-cloudinit simplifies the provisioning process through the use of a script or cloud-config document.

To get started, copy `user-data.sample` to `user-data` and make any necessary modifications.
Check out the [coreos-cloudinit documentation][coreos-cloudinit] to learn about the available features.

[coreos-cloudinit]: https://github.com/coreos/coreos-cloudinit

#### Configuration

The Vagrantfile will parse a `config.rb` file containing a set of options used to configure your CoreOS cluster.
See `config.rb.sample` for more information.

## Cluster Setup

Launching a CoreOS cluster on Vagrant is as simple as configuring `$num_instances` in a `config.rb` file to 3 (or more!) and running `vagrant up`.
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
