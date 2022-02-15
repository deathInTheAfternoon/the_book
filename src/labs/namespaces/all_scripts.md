# All scripts

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

echo "Creating single smart cables"
# red2blue means plugged into red computer with blue computer at remote end.
ip link add red2blue type veth peer name blue2red  

echo "Connecting both smart cable plugs into each computer"
ip link set red2blue netns red
ip link set blue2red netns blue

echo "Turn on NIC in each plug "
ip netns exec red ip link set red2blue up
ip netns exec blue ip link set blue2red up

echo "Assign 2 IPs, one for each embedded NIC"
ip netns exec red ip address add 192.168.1.57 dev red2blue
ip netns exec blue ip address add 192.168.1.75 dev blue2red

# Without this you see 'Network is unreachable' as IP cannot be mapped to correct cable
echo "Allow IP stack to map dest IP to smart plug"
ip netns exec red ip route add 192.168.1.75 dev red2blue
ip netns exec blue ip route add 192.168.1.57 dev blue2red

echo "Radar pings"
ip netns exec red ping -c1 192.168.1.75
ip netns exec blue ping -c1 192.168.1.57

ip netns del red
ip netns del blue
```

```bash
#!/bin/bash

if [ $(id -u) -ne 0 ]
then
    echo Please run as privileged user.
    exit 1
fi

echo "Creating computers red, green, blue"
ip netns add red
ip netns add green
ip netns add blue

echo "Creating 3 smart cables"
# red2blue means plugged into red computer with blue computer at remote end.
ip link add red2green type veth peer name green2red
ip link add green2blue type veth peer name blue2green
ip link add blue2red type veth peer name red2blue 

echo "Connecting each smart cable into each computer"
# Plug into comptuer
ip link set red2green netns red
ip link set red2blue netns red
ip link set green2red netns green
ip link set green2blue netns green
ip link set blue2green netns blue
ip link set blue2red netns blue

echo "Turn on 6 plugs"
ip netns exec red ip link set red2green up
ip netns exec red ip link set red2blue up
ip netns exec green ip link set green2red up
ip netns exec green ip link set green2blue up
ip netns exec blue ip link set blue2green up
ip netns exec blue ip link set blue2red up

echo "Assign 6 IPs to NIC in each plug"
ip netns exec red ip address add 192.168.1.56 dev red2green
ip netns exec red ip address add 192.168.1.57 dev red2blue
ip netns exec green ip address add 192.168.1.65 dev green2red
ip netns exec green ip address add 192.168.1.67 dev green2blue
ip netns exec blue ip address add 192.168.1.76 dev blue2green
ip netns exec blue ip address add 192.168.1.75 dev blue2red

# Without this you see 'Network is unreachable' as IP cannot be mapped to correct cable
echo "Allow IP table to map dest IP (only one dest IP per computer) to local smart plug"
ip netns exec red ip route add 192.168.1.65 dev red2green
ip netns exec red ip route add 192.168.1.75 dev red2blue
ip netns exec green ip route add 192.168.1.56 dev green2red
ip netns exec green ip route add 192.168.1.76 dev green2blue
ip netns exec blue ip route add 192.168.1.67 dev blue2green
ip netns exec blue ip route add 192.168.1.57 dev blue2red

# Just a loop to ping each smart cable from both directions.
echo "Ping-a-ling"
map_red=(192.168.1.65 192.168.1.75)
map_green=(192.168.1.56 192.168.1.76 )
map_blue=(192.168.1.57 192.168.1.67)

iter() {
  for array in ${!map_@}; do
    echo "Within ${array#map_}"
    declare -n cur_array="$array"
    for key in "${!cur_array[@]}"; do
	  if ip netns exec ${array#map_} ping -c1 -w2 ${cur_array[$key]} &> /dev/null
		then
			echo "${array#map_} reached IP ${cur_array[$key]}"
		else
			echo "${array#map_} FAILED to reach IP ${cur_array[$key]}"
		fi
    done
  done
}
iter

# Deleting each namespace while also delete all namespaced network paraphenalia...
ip netns del red
ip netns del green
ip netns del blue
```


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

Access to inet via GW. Some suggest single time setup is needed before running the script:
```bash
sysctl -w net.ipv4.ip_forward=1
# Don't repeat this, it keeps duplicating rules in NAT table.
iptables -t nat -A POSTROUTING ! -o br0 -s 192.168.10.0/28 -j MASQUERADE
```

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
ip netns exec red ip route add default via 192.168.10.1

ip netns exec red routel

ip link add name br0 type bridge
ip link set dev br0 up
ip link set br02red master br0
ip link set br02red up
ip address add 192.168.10.1/28 brd - dev br0

ping -c1 -w2 192.168.10.2
ip netns exec red ping -c1 192.168.10.1
ip netns exec red ping -c1 8.8.8.8

ip netns del red
ip link del br0
```

Test case exploring effect of adding different CIDR types to route table
```bash
# What happens when you 'add an IP to a bridge'?
# In the below youre not adding an IP to the bridge, your adding a NIC with a MAC and IP that is cabled to the Layer 2 switch.
# BUG: 'routel' is a bash script supplied with iproute2 that pretty prints 'ip route list' output. The version on the repo (5.5) can cause the following
# "/usr/bin/routel: 48: shift: can't shift that many". This is 'shifting' command line args. Newer version (5.16) uses a Python script but it's not yet on Ubuntu repo.

echo -e "\nAdd IP to bridge and examine route tables...$(date +"%Y-%m-%d %H:%M:%S,%3N")\n"

echo "1: ip address add 192.168.1.11 dev br0"
ip link add name br0 type bridge
ip link set dev br0 up
ip address add 192.168.1.11 dev br0
echo "192.168.1.11 added to NIC. Checking host route table..."
routel
ip link del br0

echo "2: ip address add 192.168.1.11/24 dev br0"
ip link add name br0 type bridge
ip link set dev br0 up
ip address add 192.168.1.11/24 dev br0
echo "192.168.1.11/24 added to NIC. Checking host route table..."
routel
ip link del br0

echo "3: ip address add 192.168.1.11 brd + dev br0"
ip link add name br0 type bridge
ip link set dev br0 up
ip address add 192.168.1.11 brd + dev br0
echo "192.168.1.11 added to NIC. Checking host route table..."
routel
ip link del br0

echo "4: ip address add 192.168.1.21/28 brd + dev br0"
ip link add name br0 type bridge
ip link set dev br0 up
ip address add 192.168.1.0/28 brd + dev br0 # 'brd +' will set lower 4 bits to 1 giving .15
routel
ip link del br0

echo "5: ip address add 192.168.1.20/28 brd - dev br0"
ip link add name br0 type bridge
ip link set dev br0 up
ip address add 192.168.1.255/28 brd - dev br0 # 'brd -'  will set lower 4 bits to 0 giving .240
routel
ip link del br0
```
