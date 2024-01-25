# ansible-qemu-kvm

This is a forked and updated version originally crafted by [noahbailey /ansible-qemu-kvm](https://github.com/noahbailey/ansible-qemu-kvm) to work with Ubuntu 22.04 and Debian 11. 


# Introduction
Ansible role to provision virtual machines using the QEMU and KVM systems. 

Think of it like _OpenStack for poor people._

Currently, this role only creates virtual machines running the lite version of Ubuntu 18.04. Future versions may expand on this, but it is the primary server operating system that I use. 

## Usage

Add this role to the `roles` directory in your ansible project. 

Then, include the role using a top level playbook: 

```yaml
- name: KVM hosts 
  hosts: kvm-hosts
  become: true 
  roles: 
    - ansible-qemu-kvm
```



## Variables

This role requires these variables to exist in inventory: 

#### 1. Users

```yaml
users:
- name: juan
  full_name: Juan
  passwd: $6$rounds=4096$lvnh8XZ/...
  pub_key: "ssh-ed25519 SOME LONG LETTERS"
```

This is a list of users that should be added to the server during provisioning. 

The `passwd` and `pub_key` parameters take your hashed password and your SSH public key so that you are able to access the server after provisioning. 

To generate a new password hash, use `mkpasswd -m sha-512 -R 2048` (Included in the whois package). 

#### 2. VMs
```yaml
virtual_machines:
 - name: name1
   cpu: 1
   mem: 1024
   disk: 10G
   bridge: virbr1
   #linux_distro: ubuntu20.10
   linux_distro: debian11
   net:
     ip: 192.168.200.254/24
     gateway: 192.168.200.1
     domain: local
     dns:
       - 192.168.158.1

```

The variable 'linux_distro' specifies the distribution to be installed. Currently these are supported:
- ubuntu20.10 (Actually it will install Ubuntu 22.04)
- debian11 Installs Debian 11

This is a list of virtual machines that should be built. 

Bridge: this is the name of the bridge device that references the vlan the server will be connected to. This must already exist on the KVM host. 

The virtual machine can also be built with a static IP address: 

This variable takes exactly one IP address, domain, and gateway; and two or more DNS servers. 


#### Default Values

If no value is supplied, the default settings will be used: 

* CPU: 1 core
* Memory: 512 MB
* Disk: 5GB .qcow image
* Bridge: Default libvirt NAT network 
* Network: DHCP

## How it works

This role works by downloading the Ubuntu 22.04 cloud image, then converting it to Copy-On-Write format and cloning it for each virtual machine defined. 

When VMs are launched, they are given a small secondary disk image that includes a `cloud-config` file. This file is generated from a template that includes all the users and system parameters that are defined in inventory. There is also a network-config file that contains network metadata in Netplan-V2 format. 


Before running this, ensure that networking is correctly configured on the KVM hosts. 

# Configuring the virtual network in the host (route mode) 
Before running the ansible playbook it is necesary to create a new virtual network, currently this needs to be done by hand.

## Configuration of the route virtual network
Create a file called casa.xml with the following text:

```xml

<network connections='1'>
  <name>casa</name>
  <forward dev='enps' mode='route'>
    <interface dev='enps'/>
  </forward>
  <bridge name='virbr1' stp='on' delay='0'/>
  <mac address='52:54:00:d8:b0:23'/>
  <domain name='casa'/>
  <ip address='192.168.200.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.200.128' end='192.168.200.254'/>
    </dhcp>
  </ip>
</network>
```
Yo may need to cha nge 'enps' for the actual name of your host ethernet interface.

## Define the specification 
Using *virsh* (a libvirt utility)
```bash
virsh net-define casa.xml
```
## Activate the network with:

```bash
virsh net-start casa
```

This automatically creates a bridged interface!


## Enable the network in autostart mode:

```bash
virsh net-autostart casa
```

Now, it should be enough to create the new virtual machines.

## For editing the network do:

``` bash
virsh net-edit casa
```

## Port forwarding
This is not available in the level of abstraction as virsh. So, we need to define a specific IPtables rule.

We need to create a qemu hook file in: /etc/libvirt/hooks/qemu

I found this information [here.](https://wiki.libvirt.org/Networking.html#Forwarding_Incoming_Connections)

``` bash
#!/bin/bash

# IMPORTANT: Change the "VM NAME" string to match your actual VM Name.
# In order to create rules to other VMs, just duplicate the below block and configure
# it accordingly.
if [ "${1}" = "VM NAME" ]; then

   # Update the following variables to fit your setup
   GUEST_IP=
   GUEST_PORT=
   HOST_PORT=

   if [ "${2}" = "stopped" ] || [ "${2}" = "reconnect" ]; then
    /sbin/iptables -D FORWARD -o virbr0 -p tcp -d $GUEST_IP --dport $GUEST_PORT -j ACCEPT
    /sbin/iptables -t nat -D PREROUTING -p tcp --dport $HOST_PORT -j DNAT --to $GUEST_IP:$GUEST_PORT
   fi
   if [ "${2}" = "start" ] || [ "${2}" = "reconnect" ]; then
    /sbin/iptables -I FORWARD -o virbr0 -p tcp -d $GUEST_IP --dport $GUEST_PORT -j ACCEPT
    /sbin/iptables -t nat -I PREROUTING -p tcp --dport $HOST_PORT -j DNAT --to $GUEST_IP:$GUEST_PORT
   fi
fi
```

`


## Important documentation
The configuration for the network is located in: (roles/ansible-qemu-kvm/tasks/vm.yml)
