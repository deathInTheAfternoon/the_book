# 3 computers (point-point)

The same steps as before. But a lot more work to get 3 computers connected to each other. After this we'll only use a bridge as a hub between computers. 

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

# Deleting each namespace while also delete all above network paraphenalia
ip netns del red
ip netns del green
ip netns del blue

```