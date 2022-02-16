# Rust HTTP server
We will start by building a simple HTTP server in Rust using the Rocket library. The only REST endpoint we implement returns a unique identifer (UUID). Later on, this will allow us to demonstrate round-robin load balancing is working.

The only configuration we provide is a bind address. 'Binding to an IP address' means an application starts listening for incoming messages from the network interface labelled with that IP. For example, the default bind address of the Rocket framework is 127.0.0.1 the loopback interface. This default is fine when client and server run on the same host as traffic can bounce between them without leaving the host. It also works with multiple servers as long as they bind to the same address but different port numbers (two processes cannot bind to the same port).

But a network namespace acts as an isolated host. So running the server within a Docker container but binding to 127.0.0.1 would mean the server could only accept traffic originating from within the container. So in this circumstance we need to instruct the server to bind to a different network interface, one that can listen to messages from external sources. To achive that the server shall accept a command line argument which we will process using the Clap library.

But exaclty what address should we provide? Given we don't yet know the actual IP address of the interface within the namespace, we will use the meta-address 0.0.0.0. This value isn't an address per se, but is used to signal 'bind to all addresses on this host' (in routing tables it is used to match any IP whatsoever). So if the daemon listens on this IP it will pick up any messages directed to its container. This works well in container-land because we tend to run a single application per container.

### Demo to show Docker commands to investigate bridges.
TODO: ??



# Code for Rust HTTP daemon
This project assumes you've used Cargo to create a barebone workspace (TODO: Cover workspace setup somewhere). 

The first we will add are the dependencies to the applications Cargo.toml file:
```yaml
[dependencies]
lib_container_runtime = {path = "../lib_container_runtime"}
rocket = "0.5.0-rc.1"
uuid = { version = "0.8", features = ["serde", "v4"] }
clap = {version = "3.0.14", features = ["derive"]}
```
- The lib_container_runtime is the custom Rust library created by this package.
- rocket is the lightweight HTTP server framework we shall use.
- uuid is a library for creating the UUID we will use to label each instance.
- clap is the command line argument parser we use to read the bind address and port number.

The main application is here:
```rust
#[macro_use] extern crate rocket;
use uuid::Uuid;
use clap::Parser;
use rocket::State;

// CLAP command line handling...
/// Our amazingly simple Rocket daemon
#[derive(Parser, Debug)]
#[clap(name("Rocket daemon"), author("DeathInTheAfternoon"), version("0.1"), about("Does what is says on the tin."), long_about=None)]
struct Args {
    /// Set the port on which we will listen
    /// 
    /// This parameter will configure the port on which this daemon listens for incoming requests.
    #[clap(short('p'), long("port"), default_value_t = 8080)]
    port: i32,

    /// Set bind IP on which we will listen
    /// 
    /// When in container bind to ALL interfaces 0.0.0.0
    #[clap(short('a'), long("address"), default_value = "127.0.0.1")]
    address: String,
}

// Structure used to store state shared between Rocket handlers...
struct DaemonState {
    uuid: Uuid,
}

#[get("/")]
fn index(uuid_state: &State<DaemonState>) -> String {
    format!("My id is {}", uuid_state.uuid)
}

#[rocket::main]
async fn main() {
    // Parse command line arguments using Clap
    let args = Args::parse(); 
    // Start Rocket using the custom ip address and port number, if any
    let figment = rocket::Config::figment().merge(("port", args.port))
                                                    .merge(("address", args.address));
    rocket::custom(figment)
            .mount("/", routes![index])
            .manage(DaemonState { uuid: Uuid::new_v4(), }) // Generate a uuid for this instance
            .launch().await;
}
```
The 'struct Args' is used by Clap to define and parse command line arguments. The Rust Doc '///' comments will generate --help messages.

It is not straightforward to share state in Rust. That's because of the robust concurrency decision making it encourages. So we use the mechanism provided by Rocket to hold state between handler invocations. So the 'DaemonState' structure holds shared state, in this case the UUID created when the daemon starts. You can see how state is passed into handler by examining the arguments to the 'index()' handler. You only need to add this argument for Rocket to starting passing in shared state.

## Build and run
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
Again, from another terminal (we're starting another instance so UUID will change):
```console
# Both of the following should work
curl 127.0.0.1:8081
    My id is eb84b536-b75b-4bd6-99ee-3c0066ec2943
curl 0.0.0.0:8081
    My id is eb84b536-b75b-4bd6-99ee-3c0066ec2943
```

# Switch to docker-compose
As we're going to be creating two instances of this server and a load balancing Envoy proxy, we can proceed far more quickly by using a single docker-compose file to handle the creation and shutdown of our system. Else typing 'docker run....' commands will get tedious very quickly.

Here is the first part of our docker-compose which launches two instances of the HTTP server:
```yaml
version: "3.8"

services:
  daemon1:
    # This is how you set your DNS name as well as container name. We can now use daemon1 as an envoy cluster endpoint.
    container_name: daemon1
    image: nt
    ports:
      - 8080:8080
    command: ["-a", "0.0.0.0", "-p", "8080"]
  daemon2:
    container_name: daemon2
    image: nt
    ports:
      - 8081:8081
    command: ["-a", "0.0.0.0", "-p", "8080"]

# This is how you set your own network name. Default network name is '<project-folder>_default'.
networks:
  default:
    name: daemon_network
```

Docker will create a network for these containers. This is a Linux bridge. The host network interface into this bridge is called daemon_network.
```bash
docker network ls
    NETWORK ID     NAME             DRIVER    SCOPE
    fb0133775f90   bridge           bridge    local
    ac57f940ca3d   daemon_network   bridge    local
    53238165f9e4   host             host      local
    da80602fca1b   none             null      local
```
The above command contains Docker's name for the Linux bridge. From the host let's list current bridges:
```bash
ip link show type bridge
    393: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default 
        link/ether 02:42:2f:6b:21:7d brd ff:ff:ff:ff:ff:ff
    405: br-ac57f940ca3d: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default 
        link/ether 02:42:40:37:18:5a brd ff:ff:ff:ff:ff:ff
```
The default Docker 'bridge' is implemented using Linux bridge 'docker0'. While our custom Docker bridge 'daemon_network' with id 'ac57f940ca3d' is implemented using Linux bridge 'br-ac57f940ca3d'.

We can now use Linux commands to see how the underlying bridge is configured. Let's see what VETHs are plugged into the bridge:
```bash
ip link show master br-ac57f940ca3d
407: vethe911bb8@if406: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-ac57f940ca3d state UP mode DEFAULT group default 
    link/ether 7a:06:e8:da:c4:5b brd ff:ff:ff:ff:ff:ff link-netnsid 2
409: veth7ff3490@if408: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-ac57f940ca3d state UP mode DEFAULT group default 
    link/ether f6:3b:d8:0c:13:f8 brd ff:ff:ff:ff:ff:ff link-netnsid 0
411: veth3589aa2@if410: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-ac57f940ca3d state UP mode DEFAULT group default 
    link/ether 8a:72:30:e6:17:74 brd ff:ff:ff:ff:ff:ff link-netnsid 1
```
Remember, a 'cable' between a network interface and a bridge is essentially a VETH pair. The line numbers above show where the other VETH pair is connected. For example, '411: veth3589aa2@if410' means the veth on line 411 is connected to line 410 within a namespace called '1'.
The second line is the cable running from the host to the bridge.

Let's login to one of the docker daemon's and examine the veth table (redacted):
```bash
ip link show
    ... etc ...
    410: eth0@if411: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default 
        link/ether 02:42:ac:14:00:04 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```
Clearly, this link connects the isolated eth0 to namespace '0' which is the host.

TODO: diagram

### DNS
In the above, Docker will setup the local dns resolve file as follows:
```bash
cat /etc/resolv.conf 
    nameserver 127.0.0.11
    options ndots:0
```
This nameserver is the local DNS resolver that Docker provides each container. It's job is to resolve DNS lookup by contacting the embedded DNS server that Docker runs on the host. Docker manual says:
> By default, a container inherits the DNS settings of the host, as defined in the /etc/resolv.conf configuration file. Containers that use the default bridge network get a copy of this file, whereas containers that use a custom network use Docker’s embedded DNS server, which forwards external DNS lookups to the DNS servers configured on the host.

TODO: see https://stackoverflow.com/questions/44724497/what-is-overlay-network-and-how-does-dns-resolution-work

