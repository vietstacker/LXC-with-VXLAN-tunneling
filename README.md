# LXC-with-VXLAN-tunneling

## Seting up

- Creating 2 KVM virtual machines (host1 and host2, each one them has kernel: 3.19.0-64-generic) and attached them in the network 192.168.10.0/24
- Install LXC on each virtual machine

- Interface of host1: 192.168.10.156
- Interface of host2: 192.168.10.222

The below is setting up of host2. On the host1, you should do the same step but need to change:

- The ip address of vxbr. It can be 10.1.2.1/24
- The ip of remote_ip in VXLAN creation. Now it is the IP of host2: 192.168.10.222

```html 

# Install OVS
apt-get update && sudo apt-get install -y --no-install-recommends openvswitch-switch

# Create vxbr bridge inside OVS
ovs-vsctl add-br vxbr

# Assign an IP to the vxbr interface
ip addr add 10.1.2.2/24 dev vxbr

# Add port vxlan to the bridge. It is possible since Linux kernel supports VXLAN and note that it is 
# only for UNICAST in this experiment.

ovs-vsctl add-port vxbr vxlan -- set interface vxlan type=vxlan options:remote_ip=192.168.10.156

# Add vxbr interface to the lxcbr0 bridge that is the default bridge when LXC containers are attached

brctl addif lxcbr0 vxbr

# Check the information of lxcbr0

root@host1:~# brctl show

bridge name	bridge id		STP enabled	interfaces
lxcbr0		8000.ca7db23bd844	no		vxbr

# Check the config of ubuntu02 LXC container that is running on host2

root@host2:~#  vim /var/lib/lxc/ubuntu02/config

# Network configuration
lxc.network.type = veth
lxc.network.flags = up
lxc.network.hwaddr = 00:16:3e:75:06:09
lxc.network.mtu = 1500
lxc.network.name = eth0
lxc.network.link = lxcbr0


# Check information of ovs bridges

root@host2:~# ovs-vsctl show
5c91315f-07c0-4c77-a8bd-04d8748ad07f
    Bridge vxbr
        Port vxlan
            Interface vxlan
                type: vxlan
                options: {remote_ip="192.168.10.156"}
        Port vxbr
            Interface vxbr
                type: internal
    ovs_version: "2.0.2"

```

## Ping result
- From LXC container on host2 to LXC container on host1
```html
PING 10.0.3.221 (10.0.3.221) 56(84) bytes of data.
64 bytes from 10.0.3.221: icmp_seq=1 ttl=64 time=4.24 ms
64 bytes from 10.0.3.221: icmp_seq=2 ttl=64 time=1.21 ms
64 bytes from 10.0.3.221: icmp_seq=3 ttl=64 time=0.408 ms
64 bytes from 10.0.3.221: icmp_seq=4 ttl=64 time=1.20 ms
```
## TCPDump result through the interface where two virtual machines are attached
```html
16:41:17.742019 ARP, Request who-has 192.168.10.1 tell 192.168.10.222, length 28
16:41:17.742070 ARP, Reply 192.168.10.1 is-at 52:54:00:07:99:67, length 28
16:41:20.158995 ARP, Request who-has 192.168.10.1 tell 192.168.10.156, length 28
16:41:20.159037 ARP, Reply 192.168.10.1 is-at 52:54:00:07:99:67, length 28
16:41:25.767282 IP 192.168.10.222.68 > 192.168.10.1.67: BOOTP/DHCP, Request from 52:54:00:07:93:d7, length 300
16:41:28.077613 IP 192.168.10.156.68 > 192.168.10.1.67: BOOTP/DHCP, Request from 52:54:00:78:ef:01, length 300
16:41:32.450511 IP 192.168.10.222.68 > 192.168.10.1.67: BOOTP/DHCP, Request from 52:54:00:07:93:d7, length 300
16:41:44.588203 IP 192.168.10.156.68 > 192.168.10.1.67: BOOTP/DHCP, Request from 52:54:00:78:ef:01, length 300
```
