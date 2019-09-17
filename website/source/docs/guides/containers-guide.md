---
name: 'Consul with Containers'
content_length: 15
id: containers-guide
layout: content_layout
products_used:
  - Consul
description: HashiCorp provides an official Docker image for running Consul and this guide demonstrates its basic usage.
level: Implementation
---

# Consul with Containers

In this guide, you will learn how to deploy two, joined Consul agents each running in separate Docker containers. You will also register a service and perform basic maintenance operations. The two Consul agents will form a small cluster. 

By following this guide you will learn how to:

1. Get the Docker image for Consul
1. Configure and run a Consul server
1. Configure and run a Consul client
1. Interact with the Consul agents
1. Perform maintenance operations (backup your Consul data, stop a Consul agent, etc.)

The guide is Docker-focused, but the principles you will learn apply to other container runtimes as well. 

!> Security Warning This guide is not for production use. Please refer to the [Consul Reference Architecture](https://learn.hashicorp.com/consul/datacenter-deploy/reference-architecture) for Consul best practices and the [Docker Documentation](https://docs.docker.com/) for Docker best practices.

## Prerequisites

### Docker

You will need a local install of Docker running on your machine for this guide. You can find the instructions for installing Docker on your specific operating system [here](https://docs.docker.com/install/).

### Consul (Optional)

If you would like to interact with your containerized Consul agents using a local install of Consul, follow the instructions [here](https://www.consul.io/docs/install/index.html) and install the binary somewhere on your PATH.

## Get the Docker Image

First, pull the latest image. You will use Consul's official Docker image in this guide.

```sh
$ docker pull consul
```

Check the image was downloaded by listing Docker images that match `consul`.

```sh
$ docker images -f 'reference=consul'
REPOSITORY     TAG      IMAGE ID        CREATED             SIZE
consul         latest   c836e84db154     4 days ago         107MB
```
## Configure and Run a Consul Server

Next, you will use Docker command-line flags to start the agent as a server, configure networking, and bootstrap the cluster when one server is up.

```sh
$ docker run \
    -d \
    -p 8500:8500 \
    -p 8600:8600/udp \
    --name=badger \
    consul agent -server -ui -node=server-1 -bootstrap-expect=1 -client=0.0.0.0
```

Since you started the container in detached mode, `-d`, the process will run in the background. You also set port mapping to your local machine as well as binding the cient interface of our agent to 0.0.0.0. This allows you to work directly with the Consul cluster from your local machine and to access Consul's UI and DNS over localhost. Finally, you are using Docker's default bridge network.

Note, the Consul Docker image sets up the Consul configuration directory at `/consul/config` by default. The agent will load any configuration files placed in that directory. 

~> The configuration directory is **not** exposed as a volume and will not persist data. Consul uses it only during startup and does not store any state there. 

Note, the Consul Docker image sets up the Consul configuration directory at `/consul/config` by default. You can configure Consul by placing config files in that directory.

To avoid mounting volumes or copying files to the container you can also save [configuration JSON](https://www.consul.io/docs/agent/options.html#configuration-files) to that directory via the environment variable `CONSUL_LOCAL_CONFIG`.

~> The configuration directory is **not** exposed as a volume and will not persist data across container restarts. However, Consul only reads from that directory and does not store any state to it.

### Discover the Server IP Address

You can find the IP address of the Consul server by executing the `consul members` command inside of the `badger` container. 

```sh
$  docker exec badger consul members
Node       Address         Status    Type    Build  Protocol  DC   Segment
server-1  172.17.0.2:8301  alive     server  1.4.4  2         dc1  <all>
```

## Configure and Run a Consul Client

Next, deploy a containerized Consul client and instruct it to join the server by giving it the server's IP address. Do not use detached mode, so you can reference the client logs during later steps. 

```sh
$ docker run \
   --name=fox \
   consul agent -node=client-1 -join=172.17.0.2
==> Starting Consul agent...
==> Joining cluster...
    Join completed. Synced with 1 initial agents
==> Consul agent running!
           Version: 'v1.4.4'
           Node ID: '4b6da3c6-b13f-eba2-2b78-446ffa627633'
         Node name: 'client-1'
        Datacenter: 'dc1' (Segment: '')
            Server: false (Bootstrap: false)
       Client Addr: [127.0.0.1] (HTTP: 8500, HTTPS: -1, gRPC: -1, DNS: 8600)
      Cluster Addr: 172.17.0.4 (LAN: 8301, WAN: 8302)
           Encrypt: Gossip: false, TLS-Outgoing: false, TLS-Incoming: false

```

In a new terminal, check that the client has joined by executing the `consul members` command again in the Consul server container. 

```sh
$  docker exec badger consul members
Node      Address          Status  Type    Build  Protocol  DC   Segment
server-1  172.17.0.2:8301  alive   server  1.4.3  2         dc1  <all>
client-1  172.17.0.3:8301  alive   client  1.4.3  2         dc1  <default>

```

Now that you have a small cluster, you can register a service and 
perform maintenance operations. 

## Register a Service

Start a service in a third container and register it with the Consul client. The basic service increments a number every time it is accessed and returns that number. 

Pull the container and run it with port forwarding so that you can access it from your web browser by visiting [http://localhost:9001](http://localhost:9001).

```sh
$ docker pull hashicorp/counting-service:0.0.2
$ docker run \
   -p 9001:9001 \
   -d \
   --name=weasel \
   hashicorp/counting-service:0.0.2
```

Next, you will register the counting service with the Consul client by adding a service definition file called `counting.json` in the directory `consul/config`.

```sh
$ docker exec fox /bin/sh -c "echo '{\"service\": {\"name\": \"counting\", \"tags\": [\"go\"], \"port\": 9001}}' >> /consul/config/counting.json"
```

Since the Consul client does not automatically detect changes in the 
configuration directory, you will need to issue a reload command for the same container.

```sh
$ docker exec fox consul reload
Configuration reload triggered
```

If you go back to the terminal window where you started the client, you should see logs showing that the Consul client received the hangup signal, reloaded its configuration, and synced the counting service.

```sh
2019/07/01 21:49:49 [INFO] agent: Caught signal:  hangup
2019/07/01 21:49:49 [INFO] agent: Reloading configuration...
2019/07/01 21:49:49 [INFO] agent: Synced service "counting"
```

### Use Consul DNS to Discover the Service

Now you can query Consul for the location of your service using the following dig command against Consul's DNS.

```sh
$ dig @127.0.0.1 -p 8600 counting.service.consul

; <<>> DiG 9.10.6 <<>> @127.0.0.1 -p 8600 counting.service.consul
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 47570
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 2
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;counting.service.consul.       IN      A

;; ANSWER SECTION:
counting.service.consul. 0      IN      A       172.17.0.3

;; ADDITIONAL SECTION:
counting.service.consul. 0      IN      TXT     "consul-network-segment="

;; Query time: 1 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: Tue Jul 02 09:02:38 PDT 2019
;; MSG SIZE  rcvd: 104
```

You can also see your newly registered service in Consul's UI, [http://localhost:8500](http://localhost:8500).

![Consul UI with Registered Service](/assets/images/consul-containers-ui-services.png 'Consul UI with Registered Service')

## Consul Container Maintenance Operations

### Accessing Containers

You can access a containerized Consul cluster in several different ways. 

#### Docker Exec

You can execute Consul commands directly inside of your Consul containers using `docker exec`.

```sh
$ docker exec <container_id> consul members
Node      Address          Status  Type    Build  Protocol  DC   Segment
server-1  172.17.0.2:8301  alive   server  1.5.2  2         dc1  <all>
client-1  172.17.0.3:8301  alive   client  1.5.2  2         dc1  <default>
```

#### Docker Exec Attach

You can also issue commands inside of your container by opening an interactive shell and using the Consul binary included in the container.

```sh
$ docker exec -it <container_id> /bin/sh
/ # consul members
Node      Address          Status  Type    Build  Protocol  DC   Segment
server-1  172.17.0.2:8301  alive   server  1.5.2  2         dc1  <all>
client-1  172.17.0.3:8301  alive   client  1.5.2  2         dc1  <default>
```

#### Local Consul Binary

If you have a local Consul binary in your PATH you can also export the `CONSUL_HTTP_ADDR` environment variable to point to the HTTP address of a remote Consul server. This will allow you to bypass `docker exec <container_id> consul <command>` and use `consul <command>` directly. 

```sh
$ export CONSUL_HTTP_ADDR=<consul_server_ip>:8500
$ consul members
Node      Address          Status  Type    Build  Protocol  DC   Segment
server-1  172.17.0.2:8301  alive   server  1.5.2  2         dc1  <all>
client-1  172.17.0.3:8301  alive   client  1.5.2  2         dc1  <default>
```

In this guide, you are binding your containerized Consul server's client address to 0.0.0.0 which allows us to communicate with our Consul cluster with a local Consul install. By default, the client address is bound to localhost.

```sh
$ which consul
/usr/local/bin/consul
$ consul members
Node      Address          Status  Type    Build  Protocol  DC   Segment
server-1  172.17.0.2:8301  alive   server  1.5.2  2         dc1  <all>
client-1  172.17.0.3:8301  alive   client  1.5.2  2         dc1  <default>
```

### Stopping, Starting, and Restarting Containers

The official Consul container supports stopping, starting, and restarting. To stop a container, run `docker stop`.

```sh
$ docker stop <container_id>
```

To start a container, run `docker start`.

```sh
$ docker start <container_id>
```

To do an in-memory reload, send a SIGHUP to the container.

```sh
$ docker kill --signal=HUP <container_id>
```

### Removing Servers from the Cluster

As long as there are enough servers in the cluster to maintain [quorum](/docs/internals/consensus.html#deployment-table), Consul's [autopilot](/docs/guides/autopilot.html) feature will handle removing servers whose containers were stopped. Autopilot's default settings are already configured correctly. If you override them, make sure that the following [settings](/docs/agent/options.html#autopilot) are appropriate.

* `cleanup_dead_servers` must be set to true to make sure that a stopped container is removed from the cluster.
* `last_contact_threshold` should be reasonably small, so that dead servers are removed quickly.
* `server_stabilization_time` should be sufficiently large (on the order of several seconds) so that unstable servers are not added to the cluster until they stabilize.

If the container running the currently-elected Consul server leader is stopped, a leader election will be triggered.

When a previously stopped server container is restarted using `docker start <container_id>`,  and it is configured to obtain a new IP, autopilot will add it back to the set of Raft peers with the same node-id and the new IP address, after which it can participate as a server again.


### Backing-up Data

You can back-up your Consul cluster using the [consul snapshot](https://www.consul.io/docs/commands/snapshot.html) command. 

```sh
$ docker exec <container_id> consul snapshot save backup.snap
```

This will leave the `backup.snap` snapshot file inside of your container. If you are not saving your snapshot to a [persistent volume](https://docs.docker.com/storage/volumes/) then you will need to use `docker cp` to move your snapshot to a location outside of your container.

```sh
$ docker cp <container_id>:backup.snap ./ 
```

Users running the Consul Enterprise Docker containers can run the [consul snapshot agent](https://www.consul.io/docs/commands/snapshot/agent.html) to save backups automatically. Consul Enterprise's snapshot agent also allows you to save snapshots to Amazon S3 and Azure Blob Storage.

### Environment Variables

You can add configuration by passing the configuration JSON via the environment variable `CONSUL_LOCAL_CONFIG`. 

```sh
$ docker run \
	-d \
	-e CONSUL_LOCAL_CONFIG='{
	"datacenter":"us_west",
	"server":true,
	"enable_debug":true
	}' \
	consul agent -server -bootstrap-expect=3
```

Setting `CONSUL_CLIENT_INTERFACE` or `CONSUL_BIND_INTERFACE` on `docker run` is equivalent to passing in the `-client` flag(documented [here](https://www.consul.io/docs/agent/options.html#_client)) or `-bind` flag(documented [here](https://www.consul.io/docs/agent/options.html#_bind)) to Consul on startup.

Setting the `CONSUL_ALLOW_PRIVILEGED_PORTS` runs setcap on the Consul binary, allowing it to bind to privileged ports. Note that not all Docker storage backends support this feature (notably AUFS). 

```sh
$ docker run -d --net=host -e 'CONSUL_ALLOW_PRIVILEGED_PORTS=' consul -dns-port=53 -recursor=8.8.8.8
```

## Summary

In this guide you learned to deploy a containerized Consul cluster. You also learned how to deploy a containerized service and how to configure your Consul client to register that service with your Consul cluster.

Further steps can be taken to deploy and secure a Consul cluster that is production ready, connect to other clusters in other datacenters, or deploy additional microservices that can find each other with Consul service discovery and connect securely with Consul Connect.

For additional reference documentation on Docker or HashiCorp Consul, refer to these websites:

- [Consul Documenation](https://www.consul.io/docs/index.html)
- [Docker Documentation](https://docs.docker.com/)
- [Consul @ Dockerhub](https://hub.docker.com/_/consul)
- [hashicorp/docker-consul GitHub Repository](https://github.com/hashicorp/docker-consul)