# Example code snippets

## Setup
The following examples use `com.github.docker-java:docker-java-transport-httpclient5` and
`com.github.docker-java:docker-java-core` version `3.2.5`.
See [transports](transports.md) for more information about transports.


## Connecting to the docker daemon
In this example we will connect to docker through its unix socket. You can also connect to it using the secure https socket or the 
[insecure http socket](https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-socket-option). If you are using windows,
you have to take a look at the insecure http socket.

First you have to connect to the docker api by creating a DockerClient instance. The DockerClient object is used to work with the docker
daemon. If your endoint doesn't exist or the connection is faulty for some other reason, you won't get an error before you send your first 
command. Ping the api using `pingCmd()` to ensure a successful connection.

```java
DefaultDockerClientConfig clientConfig = DefaultDockerClientConfig.createDefaultConfigBuilder()
        .withDockerHost("unix:///var/run/docker.sock")
        .withDockerTlsVerify(false)
        .withApiVersion("1.40")
        .build();

ApacheDockerHttpClient httpClient = new ApacheDockerHttpClient.Builder()
        .dockerHost(clientConfig.getDockerHost())
        .sslConfig(clientConfig.getSSLConfig())
        .build();

DockerClient client = DockerClientImpl.getInstance(clientConfig, httpClient);
client.pingCmd().exec();
``` 


## Engine information
You can infos about the docker engine using `client.infoCmd().exec()`.

```java
Info infos = client.infoCmd().exec();
``` 

The `Info` object contains information like host architecture, number of existing, running and stopped containers, resources available
to the docker engine (RAM and CPU) and many more.


## Images
**Note:** Before you create containers, you need to have the image you want to use pulled. You can do this manually using the docker cli or using the 
docker engine.

The following code snippet pulls the latest `nginx` image.

**Important:** You need to specify a version otherwise all versions will be downloaded. The method is not blocking, so you explicitly have to
wait for the image download to finish, before continuing.

```java
client.pullImageCmd("nginx:latest")     // Image name.
        .withAuthConfig(...)
        .exec(new ResultCallback<PullResponseItem>() {
    @Override
    public void onStart(Closeable closeable) {
    }

    @Override
    public void onNext(PullResponseItem object) {
        // Called many times during the pull
    }

    @Override
    public void onError(Throwable throwable) {
    }

    @Override
    public void onComplete() {
        // Called when the pull is finished
    }

    @Override
    public void close() throws IOException {
    }
});
```


## Working with containers

### List containers
You can get a list of containers using `client.listContainersCmd().exec();`
In with example all with statements are optional, if you want to filter for specific containers. We will explain a few possible filters.

```java
List<Container> containers = client.listContainersCmd()
        .withShowAll(true)          // Also show stopped containers
        .withIdFilter(Arrays.asList("id1", "id2"))  // Only return containers, which have the id 'id1' or 'id2'
        .withLabelFilter(Map...)    // Filter containers by their labels
        .withLimit(10)              // Limit the amount of returned container
        .withSince(containerId)     // Only return containers, which were created after containerId was created
        .exec();
```

Use the `withIdFilter(...)` if you want to get specific container.


### Start / stop / kill a container
**Note:** Before you can start a container, you have to create one, or you start existing containers.

```java
client.startContainerCmd(containerId).exec();   // Starts the container 'containerId'
client.stopContainerCmd(containerId).exec();    // Stops the container 'containerId'
client.killContainerCmd(containerId).exec();    // Kills the container 'containerId'
```


### Create a new container
**Note:** Make sure the images used for the containers are pulled on your host.
All `with...()` statements are optional.

#### Basic container
```java
CreateContainerResponse resp = client.createContainerCmd("nginx:latest")
        .withName("Webserver")
        .exec();
```
The `CreateContainerResponse` object contains warnings and the id of the newly created container. <br>
**Note:** The container not running by now. Use the [start container](#Start_/_stop_/_kill_a_container) 
method to start the newly created container.


### Advanced container
```java
CreateContainerResponse resp = client.createContainerCmd("nginx:latest")
        .withCmd("whoami")      // Overwrite the default cmd 
        .withEntrypoint("/usr/bin/nginx")   // Overwrites the default entrypoint
        .withLabels(Map...)     // Add labels to the container. The map key is the label name and the map value is the labels value
        .withEnv("NAME=derteufelqwe", "VERSION=1.0")    // Add env variables
        .exec();
```


### Container with volumes
The following code creates a nginx container and mounts the volume `volume_name` folder to the containers folder `/etc/nginx`.

```java
Bind bind = new Bind("/etc/nginx", new Volume("volume_name"));
HostConfig hostConfig = new HostConfig()
        .withBinds(bind);

CreateContainerResponse resp = client.createContainerCmd("nginx:latest")
        .withHostConfig(hostConfig)
        .exec();
```


### Container with bind mount
The following code creates a nginx container and mounts the hosts `/host/path` folder to the container folder `/container/path`

```java
Mount mount = new Mount()
        .withSource("/host/path")
        .withTarget("/container/path")
        .withType(MountType.BIND)
        .withReadOnly(true);

HostConfig hostConfig = new HostConfig()
        .withMounts(Collections.singletonList(mount));

CreateContainerResponse resp = client.createContainerCmd("nginx:latest")
        .withHostConfig(hostConfig)
        .exec();
```

**Note:** Binds and Mounts are a bit similar and can be exchanged. You can also use Mounts to mount a volume to a container.


### Container with exposed ports
The following code creates a nginx container, which maps the hosts port `8080` to the container port `80` for `TCP` traffic.

```java
PortBinding port = new PortBinding(
        Ports.Binding.bindPort(8080),   // The port on the docker host
        ExposedPort.tcp(80)             // The port in the container
);

HostConfig hostConfig = new HostConfig()
        .withPortBindings(port);

CreateContainerResponse resp = client.createContainerCmd("nginx:latest")
        .withHostConfig(hostConfig)
        .exec();
```


### Limit resource usage
The following code creates a nginx container, which is limited to 2 CPUs and 512 MB of RAM.

```java
HostConfig hostConfig = new HostConfig()
        .withNanoCPUs(2 * 1000000000L)      // Number of cores multiplied by 1000000000
        .withMemory(512L * 1024 * 1024);    // RAM in bytes

CreateContainerResponse resp = client.createContainerCmd("nginx:latest")
        .withHostConfig(hostConfig)
        .exec();
```

**Note:** Make sure your containers don't use more RAM than your host system can provide. This can cause your whole system to crash.
See [Docker engine and OOME]("https://docs.docker.com/config/containers/resource_constraints/#understand-the-risks-of-running-out-of-memory")


### Inspect a container
Inspecting a container gives detailed information about the container and its config.

```java
InspectContainerResponse resp = client.inspectContainerCmd(containerId)
        .exec();

// A few examples
resp.getName();     // Container name
resp.getHostConfig();   // See the previous examples for uses for the HostConfig
resp.getState();    // The state of the container like the exit code
```


### Container logs
The following code snippets downloads the logs from a container. The logs are downloaded in chunks and not line by line.
**Important:** Just like the image pull, this operation is non-blocking.

```java
client.logContainerCmd(containerId)
    .withStdOut(true)   // Download the STDOUT stream - You must set one of the streams
    .withStdErr(true)   // Download the STDERR stream
    .exec(new ResultCallback<Frame>() {
        @Override
        public void onStart(Closeable closeable) {
        }

        @Override
        public void onNext(Frame object) {
            // Gets called for every chunk
            String logText = new String(object.getPayload());
        }

        @Override
        public void onError(Throwable throwable) {
        }

        @Override
        public void onComplete() {
        }

        @Override
        public void close() throws IOException {
        }
    });
```


