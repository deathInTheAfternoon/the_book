# First smart cable (host to host)
There is always at least one namespace at all times. This is called the default or host namespace and represents your PC or laptop. But from our perspective it's just another namespace albeit one with pre-created network content.

Here we show how a smart cable is created and used in the default namespace. We create the cable, power up the NICs, assign a Layer 3 address to each MAC, create a route table entry (map destination IP to this side's MAC), then ping each side.

```bash
#!/bin/bash

if [ $(id -u) -ne 0 ]
then
    echo Please run as privileged user.
    exit 1
fi

echo "Creating one smart cable"
ip link add left type veth peer name right

echo "Turn on NICs at both ends of smart cable"
ip link set left up
ip link set right up

echo "Assign 2 IPs, one for each red plug and one for the dangling plug"
ip address add 192.168.1.57 dev left
ip address add 192.168.1.75 dev right

echo "Creating route table entries on red and host computer"
ip route add 192.168.1.75 dev left
ip route add 192.168.1.57 dev right

echo "Ping-a-ling"
ping -c1 192.168.1.75
ping -c1 192.168.1.57

# This will delete both ends of the smart cable
ip link del left
```