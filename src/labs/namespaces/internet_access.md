# Finale: internet access
We're going to create a small Layer 3 network. Each IP is within the inclusive range 192.168.10.0 - 192.168.10.15 (192.168.10.0/28). 

We shall create a single namespace with its own virtual NIC directly cabled to a Layer 2 switch. The host will be given a new virtual NIC that is also directly cabled to the same switch. We will assign the following IPs: 192.168.10.0 (broadcast), 192.168.10.2 (red NIC) and 192.168.10.1 (host NIC).

We will construct the red namespace followed by a veth pair.
```bash
ip netns add red
ip link add red2br0 type veth peer name br02red
ip link set red2br0 netns red
ip netns exec red ip link set red2br0 up
```
Next, we're going to show what happens if you add a single IP to the veth device:
```bash
ip netns exec red ip address add 192.168.10.2/32 dev red2br0
```
Because CIDR range /32 has been specified then any attempted ping from red produce the following error:
```console
Error: Nexthop has invalid gateway.
PING 192.168.10.2 (192.168.10.2) 56(84) bytes of data.

--- 192.168.10.2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1070ms

ping: connect: Network is unreachable
ping: connect: Network is unreachable
```
Running 'ip netns exec red ip address' shows a single IP in the route table:
```console
ip netns exec red ip address
307: red2br0@if306: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state LOWERLAYERDOWN group default qlen 1000
    link/ether 26:b6:8d:53:f3:b8 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.10.2/32 scope global red2br0
       valid_lft forever preferred_lft forever

# Route table looks like this
routel
    target            gateway          source    proto    scope    dev         tbl
192.168.10.2              local    192.168.10.2   kernel     host   red2br0 local
```
When you associate a single IP with a NIC, there is only enough information to add an entry in the routing table which a local loopback to the red namespace.
One way past this is to add a direct route to every device that needs to be accessed - in our case, only the host NIC connected to the other end of the bridge:
```bash
ip netns exec red ip route add 192.168.10.1 dev red2br0
```

Now a ping from red to 192.168.0.1 will work fine (i.e. 'ip netns exec red ping -c1 192.168.10.1'). It works because the routing table now contains a direct route to the host NIC:
```console
         target            gateway          source    proto    scope    dev tbl
   192.168.10.1                                                 linkred2br0 
   192.168.10.2              local    192.168.10.2   kernel     hostred2br0 local
```
## Subnets
But in reality we should use subnets in which case the route table is automatically filled out:
```bash
ip netns exec red ip address add 192.168.10.2/28 dev red2br0
```
Because the above CIDR tells us the IP of the red NIC, the broadcast address and the subnet range to which this NIC is cabled, the routing able can generate a few more entries:
```console
# This is how the command works if you specify a CIDR block
310: red2br0@if309: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state LOWERLAYERDOWN group default qlen 1000
    link/ether 1e:11:98:7b:27:1c brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.10.2/28 scope global red2br0
       valid_lft forever preferred_lft forever
#Route table
         target            gateway          source    proto    scope     dev  tbl
  192.168.10.0/ 28                    192.168.10.2   kernel     link red2br0 
   192.168.10.0          broadcast    192.168.10.2   kernel     link red2br0 local
   192.168.10.2              local    192.168.10.2   kernel     host red2br0 local
  192.168.10.15          broadcast    192.168.10.2   kernel     link red2br0 local
```
However, a routing table should contain a default address. This is the last resrot hop when a destination IP does not match any other route entries. Without this, if you 'ping 8.8.8.8' you will get an immediate fail, something like 'ing: connect: Network is unreachable'. So let us add the default route to the routing table on red:
```shell
ip netns exec red ip route add default via 192.168.10.1
```

# The full script

```bash
#!/bin/bash

if [ $(id -u) -ne 0 ]
then
    echo Please run as privileged user.
    exit 1
fi

# We're going to host in CIDR range 192.168.10.0 - 192.168.10.15 (192.168.10.0/28)
# Red NIC = 192.168.10.2. Host NIC = 192.168.10.1. Each NIC is cabled to Layer 2 switch.

ip netns add red
ip link add red2br0 type veth peer name br02red
ip link set red2br0 netns red
ip netns exec red ip link set red2br0 up
ip netns exec red ip address add 192.168.10.2/28 dev red2br0
ip netns exec red ip link
ip netns exec red ip route add default via 192.168.10.1

ip netns exec red routel

ip link add name br0 type bridge
ip link set dev br0 up
ip link set br02red master br0
ip link set br02red up
ip address add 192.168.10.1/28 brd - dev br0

#sysctl -w net.ipv4.ip_forward=1
#iptables -t nat -A POSTROUTING ! -o br0 -s 192.168.10.0/28 -j MASQUERADE

ping -c1 -w2 192.168.10.2
ip netns exec red ping -c1 192.168.10.1
ip netns exec red ping -c1 8.8.8.8

ip netns del red
ip link del br0
```
