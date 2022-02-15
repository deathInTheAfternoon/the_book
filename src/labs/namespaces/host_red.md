# Smart cable from new computer to host

Now we know how to create a single cable with both ends connected to the host, let's take one end an plug into a different device. We shall call the new computer 'red'.
The only real difference is that the user of the red computer must assign an IP from their device rather than the host operator making that assignment.

```bash
#!/bin/bash

if [ $(id -u) -ne 0 ]
then
    echo Please run as privileged user.
    exit 1
fi

echo "Creating computer red"
ip netns add red

echo "Creating one smart cable"
ip link add red2host type veth peer name host2red

echo "Connecting one end of smart cable into red. Other end stays connected to host."
ip link set red2host netns red

echo "Turn on NICs at both ends of smart cable"
ip netns exec red ip link set red2host up
ip link set host2red up

echo "Red admin assigns IP to NIC as does host admin"
ip netns exec red ip address add 192.168.1.57 dev red2host
ip address add 192.168.1.75 dev host2red

echo "Creating route table entries on red and host computer"
ip netns exec red ip route add 192.168.1.75 dev red2host
ip route add 192.168.1.57 dev host2red

echo "Ping-a-ling"
ip netns exec red ping -c1 192.168.1.75
ping -c1 192.168.1.57

# This will remove the cable and its entry in default namespace route table
ip netns del red
```
