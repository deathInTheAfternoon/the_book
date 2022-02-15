# Dockerize the server
Now we're going to wrap this application in a Docker image.

Rust containers can be large (GB) and slow to build (minutes). There are many tricks involving multi-stage builds and different images that allow you build a container that is small and quick to compile. However, we are going to do something much simpler. We're going to use the Debian 'Buster Slim' image and simply copy our Rust application folder onto that image. Then, when the image is started, we will simply run the Rust HTTP daemon.

```yaml
FROM debian:buster-slim

WORKDIR /app
COPY ./target/release .

ENTRYPOINT [ "/app/container_cli" ]
```
The base image is 69 GB. By the time this image is constructed it's around 78 MB.

I did try a Google distroless build. But this contains no tools or even a shell. So at this stage, debugging is difficult. However, once the application is robust, you can build using gcr.io/distroless/cc (cc contains a compiled libcc which Rust needs) and get down to 23 MB.

However, even buster-slim has very little in the way of utilities. So if you're debugging network issues you can easily install the following for example:
```console
apt update
apt install net-tools
apt install inetunits-ping
```

Do not build! If you build now using this dockerfile you will see a very long wait (Sending build context...) as all the files from this project's release folder are copied to the Docker daemon build context - from where the actual image is constructed. Every time you 'docker build' you will see over 2.5 GB being transferred for this simple Rust application. That's because there are a lot of compiled third-party files in 'target/deps', as well as incremental build files in 'target/build' as well as other artefacts. The way to avoid this is to use a '.dockerignore' file such as:
```yaml
# ignore folders used during release build but not required during runtime
/target/debug
/target/release/.fingerprint
/target/release/build
/target/release/deps
/target/release/incremental

.git
.gitignore
.github
.dockerignore
```
But how do you know what the .dockerignore is blocking and sending to the build context. Here's a trick. Use 'ripgrep' (the fast Rust grep tool):
```console
sudo apt-get install ripgrep
rg -uuu --ignore-file .dockerignore --files --sort path .
```
Run this from the folder containing .dockerignore and it will list exactly what is being allowed into the Docker build context.

## Build and run
```console
cargo build --release
docker build -t rust_daemon .
docker run rust_daemon --address 0.0.0.0 --port 8081
```
TODO: Check networking. However, we have not setup networking yet so connectivity is limited.