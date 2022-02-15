# Rust HTTP serve
We wish to build a simple HTTP server in Rust using the Rocket library. For demo simplicity, this system only runs on a single PC or laptop (the host).

When started, an instance of the server will generate a UUID. This is returned when the index page is requested.
Later, we will use this UUID to check round-robin load balancing is working correctly.

We will also need to bind an instance of the server to an IP address and port number. This option is useful when running inside a Docker container.
'Binding to an IP address and port number' means an application starts listening for incoming messages on its chosen IP and port. For example, the default bind address of the Rocket framework is 127.0.0.1:8080. This is generally fine if running directly on the host as multiple processes can communicate with each other without leaving the machine and relying only on the port number to distinguish themselves from each other.

## Sidebar: Linux networking namespaces
### VETH (Virtual Ethernet)
A VETH is a **local** Ethernet tunnel between two endpoints called 'devices'. Imagine attaching a NIC to a server. In Linux, a network namespace resembles an isolated sever and the VETH device is the ethernet card that will be used by that server to talk to other network namespaces (servers). It is even the case that, as with a newly installed physical card, a VETH device must be assigned an IP address so other isolated namespaces (servers) can find it.

However, installing a NIC and giving it an IP is not enough to form a network. In the physical world we build a LAN by connecting a CAT-5 cable from our NIC to a switch. If we connect a second server to that switch then we have a physical network allowing traffic to flow from one machine to another. Likewise, within a single host a Linux 'bridge' acts as a switch. The Linux kernel can create software switches on demand. 

In the software world, VETH can only create a *pair* of connected devices. Imagine if you could only buy two NICs with a cable between them. If that were the case, then you would need to install one device on a server and the other device on a different server or a physical switch to build a network. This is how VETH devices are created. So when you create your pair of devices you must associate each end with its target namespace or software switch.

### Demo to show how to create VETH devices, NS and bridge.
```console
## Create 2 namespaces, bridge and 2 VETH devices.
# Only loopback interface is created by default within namespace
ip netns add red
ip netns add blue

# Create software switch and turn it on
ip link add name br1 type bridge
ip link set br1 up

# Create a VETH pair. We name each end to show which will plugin to the server 
# and which will plugin to the switch
ip link add server-plug1 type veth peer name bridge-plug1
ip link add server-plug2 type veth peer name bridge-plug2

## Wire up the network (connect each server to the switch)
# Connect one end of the VETH pair into the server
ip link set server-plug1 netns red
ip link set server-plug2 netns blue

# Plug the other end into switch
ip link set bridge-plug1 master br1
ip link set bridge-plug2 master br1

# Assign IP to server connector. Different IPs as they will connect to the same software switch
ip netns exec red ip addr add 192.168.1.11/24 dev server-plug1
ip netns exec blue ip addr add 192.168.1.12/24 dev server-plug2

Assign IP address and broadcast address to bridge. Host traffic with destination
# range 192.168.1.10/24 will now go to this bridge. Futher, 192.168.1.10 is the IP of this bridge accessible form any namespace as well as host machine.
ip addr add 192.168.1.10/24 brd + dev br1

# Power up each connector
ip link set bridge-plug1 up
ip link set bridge-plug2 up

ip netns exec red ip link set server-plug1 up
ip netns exec blue ip link set server-plug2 up

```
TODO: This is not finished (need Gateway and NAT) see https://ops.tips/blog/using-network-namespaces-and-bridge-to-isolate-servers/

### Demo to show Docker commands to investigate bridges.

## Story continued
As we've mentioned when running within network namespace, each VETH device end will have an assigned IP address that can be used by the virtual switch to communicate with whoever is listening on that endpoint. Now if a daemon is not listening on that endpoint then a browser on the host will get no response form the HTTP daemon. This is what happens if you configure the HTTP daemon to listen on 127.0.0.1 because the bridge is sending external traffic to the other IP.

However, if the daemon listens to 0.0.0.0 it will effectively be listening on all IP addresses the container exposes. So when we start our process within a container, we will bind it to such an address.

# Code for Rust HTTP daemon
This project was created using Cargo workspaces. TODO: Cover that somewhere else.

We use these dependencies:
```yaml
[dependencies]
lib_container_runtime = {path = "../lib_container_runtime"}
rocket = "0.5.0-rc.1"
uuid = { version = "0.8", features = ["serde", "v4"] }
clap = {version = "3.0.14", features = ["derive"]}
```
- The lib_container_runtime was my custom Rust library that can be used by this package.
- rocket is a lightweight Rust HTTP server framework.
- uuid is a library for creating the UUID we will use to label each instance.
- clap is a command line argument parser we use to specify the bind address and port number.

The main application is here:
```rust
#[macro_use] extern crate rocket;
use uuid::Uuid;
use clap::Parser;
use rocket::State;

#[derive(Parser, Debug)]
#[clap(name("Rocket daemon"), author("DeathInTheAfternoon"), version("0.1"), about("Does what is says on the tin."), long_about=None)]
struct Args {
    /// Set the listening port
    /// 
    /// This parameter will configure the port on which this daemon listens for incoming requests.
    #[clap(short('p'), long("port"), default_value_t = 8080)]
    port: i32,

    /// Set host IP to which we bind
    /// 
    /// When in Docker container bind to ALL interfaces 0.0.0.0 NOT 127.0.0.1.
    #[clap(short('a'), long("address"), default_value = "127.0.0.1")]
    address: String,
}

// Structure used to store state shared between handlers...
struct DaemonState {
    uuid: Uuid,
}

#[get("/")]
fn index(hit_count: &State<DaemonState>) -> String {
    format!("My id is {}", hit_count.uuid)
}

#[get("/users")]
fn get_users() -> &'static str {
    "You're my first user"
}

#[rocket::main]
async fn main() {
    let args = Args::parse(); 
    // Start Rocket using a custom ip address and port number
    let figment = rocket::Config::figment().merge(("port", args.port))
                                                    .merge(("address", args.address));
    rocket::custom(figment)
            .mount("/", routes![index, get_users])
            .manage(DaemonState { uuid: Uuid::new_v4(), })
            .launch().await;
}
```
The 'struct Args' is used by Clap to define command line arguments. It also uses the Rust Doc '///' comments to generate help messages.

The 'DaemonState' structure holds shared state. In this case it's the UUID created when the daemon starts. Shared state is difficult to achieve in Rust as it wants to be thread safe whereas global variables make that difficult to achieve (TODO: Section on concurrency and what it expects). So instead we use the shared state system built into Rocket. In fact, you can see how the Rocket framework is passing the shared state into the 'index()' handler.

The main() function shows how to read arguments from the command line and use them to configure the rocket engine.

To build and run in the foreground (exit using Ctrl-C):
```console
cargo run --release
```
And you can check the uuid of the instance using from another terminal:
```console
curl 127.0.0.1:8080
    My id is edbf4f84-8770-4205-9fe1-70f40b0aaf2b
```
If you want to pass a command line parameter use '--' to instruct cargo:
```console
cargo run --release -- --address 0.0.0.0 --port 8081
```
Again, from another terminal (we're starting another instance to UUID will change):
```console
curl 127.0.0.1:8081
    My id is eb84b536-b75b-4bd6-99ee-3c0066ec2943
curl 0.0.0.0:8081
    My id is eb84b536-b75b-4bd6-99ee-3c0066ec2943
```