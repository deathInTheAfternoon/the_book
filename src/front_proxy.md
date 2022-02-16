# Example: front proxy load balancer
We will use an Envoy proxy to load balance across two Rust HTTP containers. Easiest if we use the pre-built Docker image.

The installation instructions are straightforward and found under [Install Envoy using Docker](https://www.envoyproxy.io/docs/envoy/v1.18.2/start/install)

You can select your own version of Envoy from a table near the bottom of that page. In this example I used the (Cloudsmith Debian repository)[https://cloudsmith.io/~tetrate/repos/getenvoy-deb-stable/setup/#formats-deb]:
```bash
curl -1sLf 'https://deb.dl.getenvoy.io/public/setup.deb.sh' | sudo -E bash
```
 I then ended up using envoy-dev (124MB) but you can use release version instead. E.g:
```bash
# Chose another version from the website
docker pull envoyproxy/envoy-dev:4e486e1d336fd0e67ea4f1ee27475daaf6291321
# Quick test: start Envoy, Crtl-C to stop
docker run --rm envoyproxy/envoy-dev:aa6a397b6536ac7949592110f5b7f76732671010
    [2022-02-16 10:32:29.097][1][info][main] [source/server/server.cc:385] initializing epoch 0 (base id=0, hot restart version=11.104)
    [2022-02-16 10:32:29.097][1][info][main] [source/server/server.cc:387] statically linked extensions:
    [2022-02-16 10:32:29.097][1][info][main] [source/server/server.cc:389]   envoy.rbac.matchers: envoy.rbac.matchers.upstream 
    ... etc
```
For now avoid the distroless build (74MB) which does not have a command line shell as during the learning/debugging process this will become indispensible.

# docker-compose 
We already have a docker-compose file that starts the two backend daemons. Let's add a section to fire up the front-proxy. This goes under the 'services' section:
```yaml
envoy:
    container_name: envoy_proxy
    image: envoyproxy/envoy-dev:4e486e1d336fd0e67ea4f1ee27475daaf6291321
    ports:
      - 10000:10000
      - 9901:9901
    volumes:
      - ./envoy/envoy-demo.yaml:/etc/envoy/envoy.yaml
```
The Envoy container is configured (below) to bind to 0.0.0.0 and to listen on two ports: 9901 and 10000. 9901 is for administrative access to the configuration web pages that Envory provides. 10000 is for incoming messages (i.e. browser traffic). Let's examine the complete front-proxy configuration file

# Envoy configuration file
```yaml
admin:
  address:
    socket_address: { address: 0.0.0.0, port_value: 9901 }

static_resources:
  listeners:
  - address:
      socket_address:
        address: 0.0.0.0
        port_value: 10000
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          codec_type: AUTO
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains:
              - "*"
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: rust_daemons
          http_filters:
          - name: envoy.filters.http.router

  clusters:
  - name: rust_daemons
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: rust_daemons
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: daemon1
                port_value: 8080
        - endpoint:
            address:
              socket_address:
                address: daemon2
                port_value: 8080
```
We wont go through every line - Envoy configuration is a fairly large topic.
The first section is self-explanatory as it simply sets up the binding address of the admin web site.

Next, let us jump to the 'clusters' section named 'rust_daemons'. This configures connections to a group of backend containers which Envoy calls an 'upstream cluster'. 'type' refers to the type of service discovery algorithm used to find the IP of each upstream container. In this case, Envoy will periodically send a DNS query via a container local resolver (usuall 127.0.0.11) which will forward the query through the container network to Docker's embedded DNS. It will use the returned IP forward messages to that container.

As an aside, Envoy can also use static discovery which just means scribing the IP addresses directly within the configuration file above. It can also use Logical service discovery. This caters for DNS round robin. This is when a repeated request for the same hostname cycles through as set of IP addresses of load balanced servers. Whereas strict DNS will reset its connections to the back end if it detects an IP change, Logical DNS will keep connections open and recycle them when a previous IP is eventually returned again. 

'load_assignment' will partition the incoming request load across endpoints. In this case, we define each endpoint by its DNS name (remember, docker-compose sets DNS name to container name) and the port on which that application is listening.

The middle section 'filter_chains' makes the connection between Envoy's listening address and the upstream cluster. This section basically says that incoming HTTP requests with a URI matching the prefix '/' (in other words all requests) will be routed to the cluster 'rust_daemons'.

## Run the system
We only need build our release HTTP daemon, construct a container, then use docker-compose to fire up (and shutdown one Envoy and two daemons):
```bash
    cargo build --release
    # If you change the container tag (nt), then also change it in docker-compose.yml
    docker build -t nt .  
    docker-compose up -d
```
You can access the admin web site using a browser at localhost:9901. 

You can see the load balancer by refreshing a browser which will return the UUID of the two daemons:
```bash
curl localhost:10000
    My id is 4db760a5-97ae-4221-8419-f5635f1b4ff4
curl localhost:10000
    My id is ddba280b-4b26-42c4-941c-e0bb89b2e17
# Repeat
curl localhost:10000
    My id is 4db760a5-97ae-4221-8419-f5635f1b4ff4
curl localhost:10000
    My id is ddba280b-4b26-42c4-941c-e0bb89b2e17
```



To shut the whole thing down:
```bash
docker-compose down
```
