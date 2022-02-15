# Labs: Namespaces

This section goes through a detailed lab covering network namespaces. Throughout the lab we will refer to a network namespace as a network computer or just computer. Of course you need far more than just a network namespace to make a fully blown container system. But we will use physical metaphors throughout. Just throw them away when more accurate concepts are fully formed.

We will also use the concept of a smart cable. Our computers do not contain NICs so they cannot talk across networks. Instead they provide a socket (aka port) into which the plug of a smart cable is inserted. The plug is the smart bit. It contains an embedded NIC. When powered up, the NIC reveals its MAC address to the host computer. Once you've done the same with a computer at the other end, you have an operational Layer 2 network. Ethernet datagrams can flow between the two NICs so a layer 2 ping will return a response from the NIC at the other end.

Of course, we don't want a life of MAC addresses of the next device. We want to assign an IP address to the NIC. This just means that every time the Layer 3 software builds an IP packet, it will use that IP address as the src IP in the header. Layer 3 will then pass this packet to Layer 2 which knows or cares nothing about those addresses. It wraps the Layer 3 pakcet in an ethernet datagram with its own header. This header contains this NICs MAC address as the src MAC in the header. 

At the other end, the NIC will notice when the dst MAC matches its own MAC, then unwrap the contents and pass it up to the Layer 3 software. 

## Our smart cable model 
When the smart cable is first used, both ends are plugged into our host machine.

One end is then inserted into another device (computer or bridge), powered up, and the user of that computer maps a Layer 3 address to the Layer 2 MAC of this NIC. The process is repeated with the other end.

We need to understand the above before we introduce bridges which are invisible to Layer 2 endpoints.

The first lab builds two newtork namespaces called red and blue.

The main references for building NS and bridge with GW to internet:
https://ops.tips/blog/using-network-namespaces-and-bridge-to-isolate-servers/
https://dev.to/polarbit/how-docker-container-networking-works-mimic-it-using-linux-network-namespaces-9mj