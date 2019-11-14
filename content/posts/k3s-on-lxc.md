---
title: "K3S on LXC"
date: 2019-10-23T15:22:01-05:00
draft: false
categories:
tags:
- docker
- kubernetes
- k3s
keywords:
summary: Setup a K3S cluster on single machine using LXC and K3S distribution.
---
Most other options for running a local K8S cluster are quiet resource intensive and aren't easy to configure and setup. I have this setup on an old computer with a Q6600 CPU and 4GB of ram. It let's me scale up the cluster to 5 worker nodes, without issues. It's no fun to have just a one node cluster.

# Host Configuration
Assuming we start with a fresh install of Debian Buster on a physical machine. 

## Neworking
### Configure Network Bridge
We'll use systemd networkd service to create the bridge network, so update `/etc/network/interfaces` to only create the loopback network interface. When we configure LXC, we'll have it use this bridge, `br0`. Each LXC guest will retrieve it's own IP from the DHCP server.

`vi /etc/network/interfaces`
{{< highlight sh>}}
# The loopback network interface
auto lo
iface lo inet loopback
{{< /highlight >}}

Create bridge device
`vi /etc/systemd/networkd/br0.netdev`
{{< highlight sh>}}
[NetDev]
Name=br0
Kind=bridge
MACAddress=<MAC address of your physical NIC>
{{< /highlight >}}

Configure bridge device to use DHCP `vi /etc/systemd/networkd/br0.network`
{{< highlight sh>}}
[Match]
Name=br0

[Network]
DHCP=ipv4
{{< /highlight >}}

Configure your physical NIC device to use the bridge `vi /etc/systemd/networkd/enp2s0.network`
{{< highlight sh>}}
[Match]
Name=<enp2s0, or whatever your NIC device name is>

[Network]
Bridge=br0
{{< /highlight >}}

Enable and start systemd-netword service with:
{{< highlight sh>}}
systemctl start systemd-networkd
systemctl enable systemd-networkd
{{< /highlight >}}

Running `networkctl` should give you this.
{{< highlight sh>}}
networkctl
#=> IDX LINK             TYPE               OPERATIONAL SETUP
#=>   1 lo               loopback           carrier     unmanaged
#=>   2 enp2s0           ether              degraded    configured
#=>   3 br0              bridge             routable    configured
{{< /highlight >}}

## Additional Host Configuration

Install lxd and some other networking tools.
{{< highlight sh>}}
apt install ebtables ethtool conntrack snapd -y
snap install lxd # install version of lxd provided by canonical through snap
{{< /highlight >}}


K3S/K8S requires that we disable swap on the system, so we disable it and comment out the entry in `/etc/fstab`. As root run:
{{< highlight sh>}}
swapoff -a
{{< /highlight >}}

Enable IP forwarding rule in `vi /etc/sysctl.conf`.
{{< highlight sh>}}
net.ipv4.ip_forward = 1
{{< /highlight >}}

Add this to crontab to run at boot with `crontab -e` as root

***The `32768` number will depend on your system configuration, so it might be different. Later on in the process if `coredns` pod fails to start, check the logs and you will see an error indicating it fails to write a value to `nf_conntrack/parameters/hashsize`. That value is what you need in the command below.***

{{< highlight sh>}}
@reboot echo "32768" > /sys/module/nf_conntrack/parameters/hashsize
{{< /highlight >}}

*REBOOT AND CHECK THE ABOVE SETTINGS ARE PERSISTENT*

## Configure LXD
Before we create any LXC containers, we have to create or update `/etc/subuid` and `/etc/subgid` files with the following entries:
{{< highlight sh>}}
root:1000000:1000000000
<your username>:1000000:1000000000
{{< /highlight >}}

Configure lxd, set it to use `br0` network bridge. Make sure to select `dir` as the storage backend. 
{{< highlight sh>}}
lxd init
{{< /highlight >}}
{{< highlight sh>}}
Would you like to use LXD clustering? (yes/no) [default=no]: 
Do you want to configure a new storage pool? (yes/no) [default=yes]:
Name of the new storage pool [default=default]: 
Name of the storage backend to use (btrfs, ceph, dir, lvm) [default=btrfs]: dir
Would you like to connect to a MAAS server? (yes/no) [default=no]:
Would you like to create a new local network bridge? (yes/no) [default=yes]: no
Would you like to configure LXD to use an existing bridge or host interface? (yes/no) [default=no]: yes
Name of the existing bridge or host interface: br0
Would you like LXD to be available over the network? (yes/no) [default=no]: yes
Address to bind LXD to (not including port) [default=all]:
Port to bind LXD to [default=8443]:
Trust password for new clients:
Again:
Would you like stale cached images to be updated automatically? (yes/no) [default=yes]:
Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]:
{{< /highlight >}}

### Create Base Container
We'll create a base container that we can later copy to start our master and worker nodes to save time and make sure they are all consistent.
{{< highlight sh>}}
lxc launch images:debian/buster kbase-buster
lxc file push /boot/config-4.19.0-6-amd64 kbase-buster/boot/config-4.19.0-6-amd64
{{< /highlight >}}

Running Docker (K3S/Kubernetes) in Docker requires changes to grant the LXC guests additional privileges. Modify container configuration with `lxc config edit kbase-buster` with the following. This will setup the AppArmor profile to allow Docker in Docker, mount `/dev/kmsg` and `/lib/modules` and make them available to the guest containers.
{{< highlight yaml>}}
config:
  raw.lxc: "lxc.apparmor.profile=unconfined\nlxc.cap.drop=\nlxc.cgroup.devices.allow=a\nlxc.mount.auto=proc:rw sys:rw cgroup:rw"
  security.nesting: "true"
  security.privileged: "true"
devices:
  devkmsg:
    path: /dev/kmsg
    source: /dev/kmsg
    type: unix-char
  libmodules:
    path: /lib/modules
    source: /lib/modules
    type: disk
{{< /highlight >}}

Next step is executed inside the container to install basic utilities. Connect to `kbase-buster` and install `apt-utils` as below.
{{< highlight sh>}}
lxc exec kbase-buster /bin/bash
apt update
apt upgrade
apt install apt-utils
{{< /highlight >}}

At this point the base container is ready. In ***part 2*** of this guide we'll install and configure a 3 node K3S cluster and deploy a simple web application.


# References
* https://github.com/corneliusweig/kubernetes-lxd
* https://github.com/charmed-kubernetes/bundle/wiki/Deploying-on-LXD
* https://ubuntu.com/blog/running-kubernetes-inside-lxd