# Provisioning

How `bastion-net` and `Bastion-srv` were created, from a clean host to a VM with a static IP and internet access.

## 1. Network

`bastion-net` is a NAT-mode libvirt network, isolated from `k3s-net` and `monitoring-net`. The bastion never gets an interface on either of those; the router is the only device that talks to all three.

```xml
<network>
  <name>bastion-net</name>
  <bridge name='virbr-bastion' stp='on' delay='0'/>
  <forward mode='nat'/>
  <ip address='10.30.0.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='10.30.0.100' end='10.30.0.200'/>
    </dhcp>
  </ip>
</network>
```

```bash
sudo virsh net-define bastion-net.xml
sudo virsh net-start bastion-net
sudo virsh net-autostart bastion-net
```

![bastion-net defined, started, and autostarted](img/bastion-net-created.png)

## 2. VM

`Bastion-srv` is cloned from the same `Alpine-base` template used for the Ansible control node in K3s-lab-monitoring, rather than installed from ISO.

```bash
sudo virt-clone \
  --original Alpine-base \
  --name Bastion-srv \
  --file /var/lib/libvirt/images/Bastion-srv.qcow2
```

`virt-clone` gave the new VM its own MAC address, but it inherited the source VM's network attachment (`monitoring-net`). Detached and reattached to the correct network:

```bash
sudo virsh detach-interface Bastion-srv --type network --mac <old-mac> --config
sudo virsh attach-interface Bastion-srv --type network --source bastion-net --model virtio --config
sudo virsh start Bastion-srv
```

![NIC moved from monitoring-net to bastion-net, VM ready](img/clone-nic-change-Bastion-srv-ready.png)

**Spec:** 1 vCPU, 1024 MB RAM, 4 GB disk — a bastion only relays SSH, it doesn't need more.

## 3. Hostname

```bash
sudo hostname Bastion-srv
echo "Bastion-srv" | sudo tee /etc/hostname
sudo sed -i 's/Alpine-base/Bastion-srv/' /etc/hosts
echo "127.0.1.1   Bastion-srv" | sudo tee -a /etc/hosts
```

![Hostname set to Bastion-srv](img/Bastion-hostname-setup.png)

## 4. Static IP

The clone came up on DHCP (`10.30.0.103`). Fixed to a static address matching the lab's `.10` convention for servers:

```
# /etc/network/interfaces
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 10.30.0.10
    netmask 255.255.255.0
    gateway 10.30.0.1
```

```bash
sudo rc-service networking restart
```

![Static IP 10.30.0.10 applied, route via 10.30.0.1](img/static-ip-setup.png)

![Gateway and internet reachable — ping to 10.30.0.1 and 8.8.8.8, DNS resolving google.com](img/static-ip-setup-done.png)

At this point `bastion-net` only has outbound internet access through libvirt's own NAT — there is no route yet to `k3s-net` or `monitoring-net`. That routing is configured on OPNsense/FortiGate, not here (see the main lab's network docs).

## 5. Base system update

```bash
sudo apk update
sudo apk upgrade
```

![System packages updated](img/update-upgrade.png)

`openssh` and `sshd` were already present and running on the cloned image — no separate install step was needed.
