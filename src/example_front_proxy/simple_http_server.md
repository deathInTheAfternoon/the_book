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
fn index(hit_count: &State<DaemonState>) -> String {
    format!("My id is {}", hit_count.uuid)
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