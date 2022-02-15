# Point to point network

In this section and the next we will adopt a convention that will save a lot of head scratching when we later examine a triple computer setup.

Remember a smart cable has a plug at either end. Our convention is to name each plug <host_computer>2<remote_computer>.
So red2blue labels the plug inserted into the red computer whose remote end is inserted into the blue computer. Of course, the plug on the other end is called blue2red.

So this smart cable is visualized red2blue<----------->blue2red.

To create the network we follow these simple steps:
- Create our computers
- Label the plugs of our smart cable as per convention
- Insert labelled plugs into correct computer
- Power up the NIC in each plug

At this point each NIC has a MAC address and we have a working Layer 2 connection between the two computers. But our applications want to communicate via Layer 4 so:
- On each computer, assign the NIC an IP address
- On each computer, map the IP of the **remote** end to the local plug/NIC (i.e. create a route table entry).

The last step is important. The crux of IP networking is the humble route table. To understand what's going on, imagine a physical PC with two NICs attached to separate networks say 10._ and 192._. Now let's say the local computer wants to talk to a remote NIC with IP 10.1.1.1. For this to succeed, the IP route table must have already been configured with two entries (e.g. 10. , eth1) that link a physical NIC with the network to which is it connected. This process is repeated but we won't cover it here. 

In our case, the local route table will link the local NIC with the exact IP address of the remote computer. Consider a ping to that remote address. The local computer will lookup the NIC connected to that computer and send a packet on its merry way. Of course, there is only a single entry in our route table. But without that entry the IP lookup will fail given a message such as 'network unreachable'.

At this point we have a working Layer 4 connection. Given the IP of the remote end, a host can lookup the local plug it must use (there's only one at the moment) to send Layer 4 packets to that remote host.
- Ping the remote end from each computer

The following bash file contents will create our point-point network and should be run as sudo.

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

# Layer 2 network
echo "Creating single smart cable red2blue<----->blue2red"
ip link add red2blue type veth peer name blue2red  

echo "Connecting either end of smart cable into each computer"
ip link set red2blue netns red
ip link set blue2red netns blue
# Or in one go: ip link add red2blue netns red type veth peer blue2red netns blue

echo "Turn on NIC in each plug "
ip netns exec red ip link set red2blue up
ip netns exec blue ip link set blue2red up

# Each NIC now has a MAC address.

#Layer 4 addressing
echo "Assign 2 IPs, one to each NIC"
ip netns exec red ip address add 192.168.1.57 dev red2blue
ip netns exec blue ip address add 192.168.1.75 dev blue2red

# Without this you see 'Network is unreachable' as IP cannot be mapped to correct cable
echo "Allow IP table to map dest IP (only one dest IP per computer) to local smart plug"
ip netns exec red ip route add 192.168.1.75 dev red2blue
ip netns exec blue ip route add 192.168.1.57 dev blue2red

echo "Radar pings"
ip netns exec red ping -c1 192.168.1.75
ip netns exec blue ping -c1 192.168.1.57

# Deleting each namespace while also delete all above network paraphenalia
ip netns del red
ip netns del blue

```
