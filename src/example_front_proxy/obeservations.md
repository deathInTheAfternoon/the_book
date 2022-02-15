# Observational notes

# Linux namespaces, bridges, NATs and Gateways!!

## Pre-req tools
```console
ip address show

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: bond0: <BROADCAST,MULTICAST,MASTER> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 12:b4:72:fc:3f:55 brd ff:ff:ff:ff:ff:ff
3: dummy0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether d2:41:67:34:c8:66 brd ff:ff:ff:ff:ff:ff
4: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:15:5d:b5:bb:1f brd ff:ff:ff:ff:ff:ff
    inet 172.24.226.207/20 brd 172.24.239.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::215:5dff:feb5:bb1f/64 scope link
       valid_lft forever preferred_lft forever
5: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
6: sit0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/sit 0.0.0.0 brd 0.0.0.0
7: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:68:ab:27:aa brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:68ff:feab:27aa/64 scope link
       valid_lft forever preferred_lft forever
```
This lists network interfaces.
bond0 allows this machine to have it's NICs combined into a single logical NIC (network bonding). This can provide fault-tolerance or increased throughput. If one of the physical NICs fails it can be replaced without distrubing traffic flow. One might also load balance across each NIC to improve throughput.
IPIP tunneling (tunl0@NONE) embeds isolated IP network messages within internet IP packets.  Simple Internet Transition tunneling (sit0@NONE) embeds isolated IPv6 networks within internet IP packets. Described here https://developers.redhat.com/blog/2019/05/17/an-introduction-to-linux-virtual-interfaces-tunnels .


## Experiments
2 MAC addressed computers
- Create 2 computers
- Create a Layer 2 link between two computers.
2 IP addressed computers
- Create a Layer 4 link between them and demonstrate ping.
3 IP addressed computers
- Create new computer
- Create 2 Layer 4 links between the other two computers
- Ping all machines from each computer
- Destroy links and computers
That's the first network. At this stage no IP packets or Ethernet fragments can escape these machines. Most importantly, they cannot communicate with the host and so they cannot access the physical network to which the host is connected (e.g. internet).

Note: we run commands from the host aka default or host namespace.
Create 2 network computers:
```console
ip netns add red
ip netns add blue
# Check
ip netns list
    red
    blue
```
Check, no smart cable has been inserted yet:
```console
ip netns exec blue ip address show
    1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
        link/ipip 0.0.0.0 brd 0.0.0.0
    3: sit0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
        link/sit 0.0.0.0 brd 0.0.0.0
ip netns exec red ip address show
    1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
        link/ipip 0.0.0.0 brd 0.0.0.0
    3: sit0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
        link/sit 0.0.0.0 brd 0.0.0.0
```
Create a Layer 2, smart-cable link between them
```console
# Create the smart cable (i.e. NIC embedded within smart socket at either end. So each end has a MAC)
ip link add vethblue type veth peer name vethred
# Connect smart cable to each NS.
ip link set vethblue netns blue
ip link set vethred netns red

# Power each end up
# Turn smart cable NICs on
ip netns exec blue ip link set vethblue up
ip netns exec red ip link set vethred up
```

Prove that the smart cable is MAC addressable:
```
ip netns exec blue ip address (redacted)
    251: vethblue@if250: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
        link/ether 5a:73:20:65:43:28 brd ff:ff:ff:ff:ff:ff link-netns red
ip netns exec red ip address
    250: vethred@if251: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
        link/ether 1a:6f:71:b2:72:ec brd ff:ff:ff:ff:ff:ff link-netns blue
```
It's quite typical that once a smart plug is connected, the line number of one end is used as the plug name on the other side.

Theoretically, we could Layer 2 ping but such software is hard to find and doesn't show much.

Now make each cable end IP addressable:
```console
ip netns exec blue ip address add 192.168.1.11/24 dev vethblue
ip netns exec red ip address add 192.168.1.12/24 dev vethred
```

Now IP packets can be sent between computers (wrapped in the ethernet fragment of Layer 2)
```console
 ip netns exec blue ping 192.168.1.12
    PING 192.168.1.12 (192.168.1.12) 56(84) bytes of data.
    64 bytes from 192.168.1.12: icmp_seq=1 ttl=64 time=1.83 ms
    64 bytes from 192.168.1.12: icmp_seq=2 ttl=64 time=0.037 ms
ip netns exec red ping 192.168.1.11
    PING 192.168.1.11 (192.168.1.11) 56(84) bytes of data.
    64 bytes from 192.168.1.11: icmp_seq=1 ttl=64 time=0.164 ms
    64 bytes from 192.168.1.11: icmp_seq=2 ttl=64 time=0.194 ms
```

Let's add Layer 4 link to a new computer and wire all three together
```console
ip netns add green
ip link add vethgreen2blue type veth peer name vethblue2green
ip link set vethgreen2blue netns green
ip link set vethblue2green netns blue

ip link add vethgeeen2red type veth peer name vethred2green
ip link set vethgreen2red netns green
ip link set vethred2green netns red

ip netns exec green ip link set vethgreen2blue up
ip netns exec green ip link set vethgreen2red up
# Ensure lo is up (otherwise it wont work)
ip netns exec green ip link set lo up

ip netns exec blue ip link set vethblue2green up
ip netns exec red ip link set vethred2green up

ip netns exec green ip address add 192.168.1.15/24 dev vethgreen2blue
ip netns exec blue ip address add 192.168.1.16/24 dev vethblue2green
ip netns exec green ip address add 192.168.1.30/24 dev vethgreen2red
ip netns exec red ip address add 192.168.1.21/24 dev vethred2green
```
We have a minor issue at this point. We won't address it in detail because we're taking this no further. But look at the route table within each namespace:
```console
sudo ip netns exec blue netstat -r
    Kernel IP routing table
    192.168.1.0     0.0.0.0         255.255.255.0   U         0 0          0 vethblue
    192.168.1.0     0.0.0.0         255.255.255.0   U         0 0          0 vethblue2green

sudo ip netns exec red netstat -r
    Kernel IP routing table
    Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
    192.168.1.0     0.0.0.0         255.255.255.0   U         0 0          0 vethred
    192.168.1.0     0.0.0.0         255.255.255.0   U         0 0          0 vethred2green

sudo ip netns exec green netstat -r
    Kernel IP routing table
    Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
    192.168.1.0     0.0.0.0         255.255.255.0   U         0 0          0 vethgreen2blue
    192.168.1.0     0.0.0.0         255.255.255.0   U         0 0          0 vethgreen2red

```
When we added the IP address to each veth smart plug, a route table entry is created for the its subnet. But given we're adding two plugs the same subnet is added twice. With two identical entries in each table only the first will be used to resolve a destination address. For example, let's try and ping the green namespace from the red:
```console
sudo ip netns exec red ping 192.168.1.30
    PING 192.168.1.30 (192.168.1.30) 56(84) bytes of data.
    From 192.168.1.12 icmp_seq=1 Destination Host Unreachable
```
The ping IP address is resolved using the first destination/genmask entry which communicates via vethblue. Hence the hoped for destination is unreachable.
We shall fix the above by simply adding an exact match for each destination in each namespace route table. This will always be matched before the subnet entry. Note: given the vethblue2red/vethred2blue pair was added first, we will let blue and red continue to use the subnet entry in their tables.
```console
sudo ip netns exec blue ip route add 192.168.1.15 dev vethblue2green
sudo ip netns exec red ip route add 192.168.1.30 dev vethred2green

sudo ip netns exec green ip route add 192.168.1.16 dev vethgreen2blue
sudo ip netns exec green ip route add 192.168.1.21 dev vethgreen2red
```
The only other issue I had at this stage was the plug vethred2green was still in the default namespace (host machine). Ping kept failing. 
Also this error 'ip netns exec red ip address add 192.168.1.113 red2green' giving 'Error: either "local" is duplicate, or "red2green" is a garbage'.
Cause: I forgot the 'dev' statement before red2green.

```console
sudo ip link list
256: vethred2green@if257: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 0a:a9:40:31:32:5c brd ff:ff:ff:ff:ff:ff link-netns green
```
So just rerun command to connect this plug to the red computer.

```console
# NOTE: we have two cables so we need to tell IP stack which interface is connected as we are not using route tables
ip netns exec blue ping -I vethblue2green 192.168.1.15
ip netns exec green ping -I vethgreen2blue 192.168.1.16
```

Also, the host knows nothing about this smart cable as it goes direct between NS blue and red:
```console
ip address
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
        valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host
        valid_lft forever preferred_lft forever
    2: bond0: <BROADCAST,MULTICAST,MASTER> mtu 1500 qdisc noop state DOWN group default qlen 1000
        link/ether 12:b4:72:fc:3f:55 brd ff:ff:ff:ff:ff:ff
    3: dummy0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default qlen 1000
        link/ether d2:41:67:34:c8:66 brd ff:ff:ff:ff:ff:ff
    4: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
        link/ether 00:15:5d:b5:bb:1f brd ff:ff:ff:ff:ff:ff
        inet 172.24.226.207/20 brd 172.24.239.255 scope global eth0
        valid_lft forever preferred_lft forever
        inet6 fe80::215:5dff:feb5:bb1f/64 scope link
        valid_lft forever preferred_lft forever
    5: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
        link/ipip 0.0.0.0 brd 0.0.0.0
    6: sit0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
        link/sit 0.0.0.0 brd 0.0.0.0
    7: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
        link/ether 02:42:68:ab:27:aa brd ff:ff:ff:ff:ff:ff
        inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
        valid_lft forever preferred_lft forever
        inet6 fe80::42:68ff:feab:27aa/64 scope link
        valid_lft forever preferred_lft forever
```

# Docker
A Docker container is given a MAC address. If you start a container and inspect the bridge to which it is connected you'll see:
```json
 "Containers": {
            "480f904510f360d02c62af8cdac115d6b5dcc2904c93e23fca1b858e3ed35c3c": {
                "Name": "gallant_mirzakhani",
                "EndpointID": "c30a9a3e517246e744545f4ae957e4d4f830cf48eafd5f6c30b79b103632f0c5",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }
        },
```
The Mac starts with 02 'locally administered' flag. This flag ensures a container's MAC will never collide with a physical NIC's MAC. Note sure what 42 flags.
But the next four hex digits are the IP address: 0xAC = 172, 0x11 = 17, 0x00, 0x02.



TODO: What is the gateway? Does host have a MAC?

Why does a bridge have an IP address (layer 3)


# IP command
Keep banging my head into this but without many explanations about what it returns.

```console
$ ip address
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: bond0: <BROADCAST,MULTICAST,MASTER> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 12:b4:72:fc:3f:55 brd ff:ff:ff:ff:ff:ff
3: dummy0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether d2:41:67:34:c8:66 brd ff:ff:ff:ff:ff:ff
4: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:15:5d:b5:bb:1f brd ff:ff:ff:ff:ff:ff
    inet 172.24.226.207/20 brd 172.24.239.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::215:5dff:feb5:bb1f/64 scope link
       valid_lft forever preferred_lft forever
5: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
6: sit0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/sit 0.0.0.0 brd 0.0.0.0
7: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:68:ab:27:aa brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:68ff:feab:27aa/64 scope link
       valid_lft forever preferred_lft forever
```
A physical interface is a wired or wireless NIC.
