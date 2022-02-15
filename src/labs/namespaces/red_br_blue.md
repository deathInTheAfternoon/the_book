# Two computers (bridged)
This will give us the same network as before but with a bridge instead of a direct connection. The main point is the bridge is transparent from the perspective of both computers.

```bash
#!/bin/bash

if [ $(id -u) -ne 0 ]
then
    echo Please run as privileged user.
    exit 1
fi

echo "Creating computers red, blue"
ip netns add red
ip netns add blue

echo "Creating two smart cables"
ip link add red2bridge type veth peer name bridge2red
ip link add blue2bridge type veth peer name bridge2blue

echo "Connecting one end of smart cable into computer. Other end dangling in default namespace."
# The other end is dangling in default (host) namespace for now
ip link set red2bridge netns red
ip link set blue2bridge netns blue

echo "Creating bridge (always in default namespace)"
ip link add name colour_bridge type bridge

echo "Connecting dangling plugs into bridge"
ip link set bridge2red master colour_bridge
ip link set bridge2blue master colour_bridge

echo "Turn on NICs and bridge"
ip netns exec red ip link set red2bridge up
ip netns exec blue ip link set blue2bridge up
# Remember, bridge and its plugs are always in default namespace
ip link set bridge2red up
ip link set bridge2blue up
ip link set colour_bridge up

echo "Assign 2 IPs, one for each plug connected to a computer"
ip netns exec red ip address add 192.168.1.57 dev red2bridge
ip netns exec blue ip address add 192.168.1.75 dev blue2bridge

# Without this you see 'Network is unreachable' as IP cannot be mapped to correct cable
echo "Creating route table entries on each computer"
ip netns exec red ip route add 192.168.1.75 dev red2bridge

ip netns exec blue ip route add 192.168.1.57 dev blue2bridge

echo "Radar pings pass through 'invisible' bridge"
echo -e "\nRed pings blue"
ip netns exec red ping -c1 192.168.1.75
echo -e "\nBlue pings red"
ip netns exec blue ping -c1 192.168.1.57

ip netns del red
ip netns del blue
#The bridge is in the host namespace so delete needed
ip link del colour_bridge
```
