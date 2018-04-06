---
description: Learn to use the built-in network debugger to debug overlay networking problems
keywords: network, troubleshooting, debug
title: Debug overlay or swarm networking issues
---

**WARNING**
This tool can change the internal state of the libnetwork API, be really mindful
on its use and carefully read the following guide. Improper use of it will damage
or permanently destroy the network configuration.


Docker CE 17.12 and higher introduce a network debugging tool designed to help
troubleshoot issues with overlay networks and swarm services running on Linux
hosts.  When enabled, a server in the engine listens on a configurable port and
provides network diagnostic information. This troubleshooting tool should only
be started to debug specific issues, and should not be left running all the
time.

Information about networks is stored in a database which can be examined using
the API exposed by this tool. Currently the database contains information about
overlay networks as well as service discovery data.

The API exposes endpoints to query and control the network debugging tool.  A
CLI integration is provided as a preview, but the implementation is not yet
considered stable and commands and options may change without notice.

The tool is available in the form of two container images:
1) client CLI only: `dockereng/network-diagnostic:onlyclient`
2) client CLI plus "docker in docker" engine: `dockereng/network-diagnostic:17.12-dind`
The latter allows use of the tool with a cluster running an engine older than 17.12.

## Enable the diagnostic server on a running Docker engine

The tool currently only works on Docker hosts running on Linux. To enable it on a node
follow the step below.

1.  Set the `network-diagnostic-port` to a port which is free on the Docker
    host, in the `/etc/docker/daemon.json` configuration file.

    ```json
    “network-diagnostic-port”: <port>
    ```

2.  Get the process ID (PID) of the `dockerd` process. It is typically a number
    from 2 to 6 digits long.

    ```bash
    $ ps h -o pid -C dockerd
    ```

3.  Reload the Docker configuration without restarting Docker, by sending the
    `HUP` signal to the PID you found in the previous step.

    ```bash
    kill -HUP <pid-of-dockerd>
    ```

If the OS uses systemd, the command `systemctl reload docker` can be used to
perform steps 2 and 3 automatically.


A message like the following will appear in the Docker engine logs:

```none
Starting the diagnostic server listening on <port> for commands
```

## Disable the diagnostic server on a running Docker engine

Repeat these steps for each node participating in the swarm.

1.  Remove the `network-diagnostic-port` key from the `/etc/docker/daemon.json`
    configuration file.

2.  Get the process ID (PID) of the `dockerd` process. It is typically a number
    from 2 to 6 digits long.

    ```bash
    $ ps h -o pid -C dockerd
    ```

3.  Reload the Docker configuration without restarting Docker, by sending the
    `HUP` signal to the PID you found in the previous step.

    ```bash
    kill -HUP <pid-of-dockerd>
    ```

A message like the following will appear in the Docker engine logs:

```none
Disabling the diagnostic server
```

## Access the diagnostic tool's API

The network diagnostic tool exposes its own RESTful API. To access the API,
send a HTTP request to the port where the tool is listening. The following
commands assume the tool is listening on port 2000.

Usage examples for some (but not all) endpoints are provided below.

### Get help

```bash
$ curl localhost:2000/help

OK
/updateentry
/getentry
/gettable
/leavenetwork
/createentry
/help
/clusterpeers
/ready
/joinnetwork
/deleteentry
/networkpeers
/
/join
```

### Join or leave the network database cluster

```bash
$ curl localhost:2000/join?members=ip1,ip2,...
```

```bash
$ curl localhost:2000/leave?members=ip1,ip2,...
```

`ip1`, `ip2`, ... are the swarm node ips (usually one is enough)

### Join or leave a network

```bash
$ curl localhost:2000/joinnetwork?nid=<network id>
```

```bash
$ curl localhost:2000/leavenetwork?nid=<network id>
```

`network id` can be retrieved on the manager with `docker network ls --no-trunc` and has
to be the full length identifier

### List cluster peers

```bash
$ curl localhost:2000/clusterpeers
```

### List nodes connected to a given network

```bash
$ curl localhost:2000/networkpeers?nid=<network id>
```
`network id` can be retrieved on the manager with `docker network ls --no-trunc` and has
to be the full length identifier

### Dump database tables

The tables are called `endpoint_table` and `overlay_peer_table`.
The `overlay_peer_table` contains all the overlay forwarding information
The `endpoint_table` contains all the service discovery information

```bash
$ curl localhost:2000/gettable?nid=<network id>&tname=<table name>
```

### Interact with a specific database table

The tables are called `endpoint_table` and `overlay_peer_table`.

```bash
$ curl localhost:2000/<method>?nid=<network id>&tname=<table name>&key=<key>[&value=<value>]
```

Note:
Create operations on tables have node ownership; this means they persist only
while the node that inserted them remains in the cluster.

## Access the diagnostic tool's CLI

The client CLI is provided as a preview and is not yet stable. Commands or
options may change at any time.

The client CLI is an executable called `diagnosticClient`.  For clusters
running 17.12 or later, it is available via the entrypoint of a "client only"
container.

`docker run --net host dockereng/network-diagnostic:onlyclient -v -net <full network id> -t sd`

The following flags are supported:

| Flag          | Description                                     |
|---------------|-------------------------------------------------|
| -t <string>   | Table one of `sd` or `overlay`.                 |
| -ip <string>  | The IP address to query. Defaults to 127.0.0.1. |
| -net <string> | The target network ID.                          |
| -port <int>   | The target port. (default port is 2000)         |
| -a            | Join/leave network                              |
| -v            | Enable verbose output.                          |

*NOTE* `-a` is not needed and should be avoided for networks associated with
active workloads on a running engine. The `-a` flag causes the engine to join
*and then leave* the specified network, which will sever any active workloads
on the engine from that network .  If `diagnoseClient` it is talking to an
engine with running containers on a particular network the `-a` flag is not
needed; `diagnoseClient` will simply dump current state.  When running
`diagnosticClient` with the `dind` container, however, the `-a` flag is
required to avoid retrieving empty results.

### Container version of the diagnostic tool

A container image providing both the client CLI and a 17.12 "dind" engine is
available for troubleshooting in clusters on versions prior to 17.12.  The
container needs to be run using privileged mode so that it can open privileged
ports and join the cluster.
*NOTE*
Remember that table operations have node ownership, so any `create entry` will
persist only so long as the diagnostic container remains in the swarm.

1.  Make sure that the node where the diagnostic client will run is not part of the swarm, if so do `docker swarm leave -f`

2.  To run the container, use a command like the following:

    ```bash
    $ docker container run --name net-diagnostic -d --privileged --network host dockereng/network-diagnostic:17.12-dind
    ```

3.  Connect to the container using `docker exec -it net-diagnostic sh`,
    and start the server using the following command:

    ```bash
    $ kill -HUP 1
    ```

4.  Join the diagnostic container as a worker to the swarm:

    ```bash
    $ docker swarm join <worker-join-token>
    ```

    `<worker-join-token>` can be found by running `docker swarm join-token worker -q` on a manager


5.  On a manager node set the diagnostic container's availability status to
    `drain` to prevent the cluster from scheduling tasks to it:


    ```bash
    $ docker node update --availability drain <nodeid>
    ```

6. Run the diagnostic CLI within the container.

    ```bash
    $ ./diagnosticClient <flags>...
    ```

7.  When finished debugging, leave the swarm and stop the container.


    ```bash
    $ docker swarm leave
    Node left the swarm.
    $ exit
    ```

8.  On a manager node, remote the diagnostic container from the swarm.


    ```bash
    $ docker node rm <nodeid>
    ```

### Examples

The following commands dump the service discovery table and verify node
ownership.

*NOTE*
Remember to use the full network ID. You can easily find that with `docker
network ls --no-trunc`.

**Service discovery and load balancer:**

```bash
$ diagnosticClient -t sd -v -net n8a8ie6tb3wr2e260vxj8ncy4 -a
```

**Overlay network:**

```bash
$ diagnosticClient -port 2001 -t overlay -v -net n8a8ie6tb3wr2e260vxj8ncy4 -a
```
