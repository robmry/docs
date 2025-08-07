---
title: Networking overview
linkTitle: Networking
weight: 30
description: Learn how networking works from the container's point of view
keywords: networking, container, standalone, IP address, DNS resolution
aliases:
- /articles/networking/
- /config/containers/container-networking/
- /engine/tutorials/networkingcontainers/
- /engine/userguide/networking/
- /engine/userguide/networking/configure-dns/
- /engine/userguide/networking/default_network/binding/
- /engine/userguide/networking/default_network/configure-dns/
- /engine/userguide/networking/default_network/container-communication/
- /engine/userguide/networking/dockernetworks/
- /network/
---

Container networking refers to the ability for containers to connect to and
communicate with each other, and with non-Docker network services.

Containers have networking enabled by default, and they can make outgoing
connections. A container has no information about what kind of network it's
attached to, or whether its network peers are also Docker containers. A
container only sees a network interface with an IP address, a gateway, a
routing table, DNS services, and other networking details.

This page describes networking from the point of view of the container,
and the concepts around container networking.

When Docker Engine on Linux starts for the first time, it has a single
built-in network called the "default bridge" network. When you run a
container with no `--network` option, it is connected to the default bridge.

Containers attached to the default bridge have access to network services
outside the Docker host. They use "masquerading" which means, if the
Docker host has Internet access, no additional configuration is needed
for the container to have Internet access.

For example, to run a container on the default bridge network, and have
it ping an Internet host:

```console
$ docker run --rm -ti busybox ping -c1 docker.com
PING docker.com (23.185.0.4): 56 data bytes
64 bytes from 23.185.0.4: seq=0 ttl=62 time=6.564 ms

--- docker.com ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 6.564/6.564/6.564 ms
```

## User-defined networks

With the default configuration, containers attached to the default
bridge network have unrestricted network access to each other using
container IP addresses. They cannot refer to each other by name.

It can be useful to separate groups of containers that should have full
access to each other, but restricted access to containers in other groups.

You can create custom, user-defined networks, and connect groups of containers
to the same network. Once connected to a user-defined network, containers
can communicate with each other using container IP addresses or container names.

The following example creates a network using the `bridge` network driver and
runs a container in that network:

```console
$ docker network create -d bridge my-net
$ docker run --network=my-net -it --name=container3 busybox
```

### Drivers

Docker Engine has a number of network drivers, as well as the default "bridge".
On Linux, the following built-in network drivers are available:

| Driver                          | Description                                                         |
|:--------------------------------|:--------------------------------------------------------------------|
| [bridge](./drivers/bridge.md)   | The default network driver.                                         |
| [host](./drivers/host.md)       | Remove network isolation between the container and the Docker host. |
| [none](./drivers/none.md)       | Completely isolate a container from the host and other containers.  |
| [overlay](./drivers/overlay.md) | Swarm Overlay networks connect multiple Docker daemons together.    |
| [ipvlan](./drivers/ipvlan.md)   | Connect containers to external VLANs.                               |
| [macvlan](./drivers/macvlan.md) | Containers appear as devices on the host's network.                 |

More information can be found in the network driver specific pages, including
their configuration options and details about their functionality.

Native Windows containers have a different set of drivers, see
[Windows container network drivers](https://learn.microsoft.com/en-us/virtualization/windowscontainers/container-networking/network-drivers-topologies).

### Connecting to multiple networks

Connecting a container to a network can be compared to connecting an ethernet
cable to a physical NIC. Just as a host can be connected to multiple ethernet
networks, a container can be connected to multiple Docker networks.

For example, a frontend container may be connected to a bridge network
with external access, and a
[`--internal`](/reference/cli/docker/network/create/#internal) network
to communicate with containers running backend services that do not need
external network access.

A container may also be connected to different types of network. For example,
an `ipvlan` network to provide internet access, and a `bridge` network for
access to local services.

Containers can also share networking stacks, see [Container networks](#container-networks).

When sending packets, if the destination is an address in a directly connected
network, packets are sent to that network. Otherwise, packets are sent to
a default gateway for routing to their destination. In the example above,
the `ipvlan` network's gateway must be the default gateway.

The default gateway is selected by Docker, and may change whenever a
container's network connections change.
To make Docker choose a specific default gateway when creating the container
or connecting a new network, set a gateway priority. See option `gw-priority`
for the [`docker run`](/reference/cli/docker/container/run.md) and
[`docker network connect`](/reference/cli/docker/network/connect.md) commands.

The default `gw-priority` is `0` and the gateway in the network with the
highest priority is the default gateway. So, when a network should always
be the default gateway, it is enough to set its `gw-priority` to `1`.

```console
$ docker run --network name=gwnet,gw-priority=1 --network anet1 --name myctr myimage
$ docker network connect anet2 myctr
```

## Published ports

When you create or run a container using `docker create` or `docker run`, all
ports of containers on bridge networks are accessible from the Docker host and
other containers connected to the same network. Ports are not accessible from
outside the host or, with the default configuration, from containers in other
networks.

Use the `--publish` or `-p` flag to make a port available outside the host,
and to containers in other bridge networks.
This creates a firewall rule in the host,
mapping a container port to a port on the Docker host to the outside world.
Here are some examples:

| Flag value                      | Description                                                                                                                                             |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `-p 8080:80`                    | Map port `8080` on the Docker host to TCP port `80` in the container.                                                                                   |
| `-p 192.168.1.100:8080:80`      | Map port `8080` on the Docker host IP `192.168.1.100` to TCP port `80` in the container.                                                                |
| `-p 8080:80/udp`                | Map port `8080` on the Docker host to UDP port `80` in the container.                                                                                   |
| `-p 8080:80/tcp -p 8080:80/udp` | Map TCP port `8080` on the Docker host to TCP port `80` in the container, and map UDP port `8080` on the Docker host to UDP port `80` in the container. |

> [!IMPORTANT]
>
> Publishing container ports is insecure by default. Meaning, when you publish
> a container's ports it becomes available not only to the Docker host, but to
> the outside world as well.
>
> If you include the localhost IP address (`127.0.0.1`, or `::1`) with the
> publish flag, only the Docker host and its containers can access the
> published container port.
>
> ```console
> $ docker run -p 127.0.0.1:8080:80 -p '[::1]:8080:80' nginx
> ```
>
> > [!WARNING]
> >
> > In releases older than 28.0.0, hosts within the same L2 segment (for example,
> > hosts connected to the same network switch) can reach ports published to localhost.
> > For more information, see
> > [moby/moby#45610](https://github.com/moby/moby/issues/45610)

Ports on the host's IPv6 addresses will map to the container's IPv4 address
if no host IP is given in a port mapping, the bridge network is IPv4-only,
and `--userland-proxy=true` (default).

For more information about port mapping, including how to disable it and use
direct routing to containers, see
[packet filtering and firewalls](./packet-filtering-firewalls.md).

## IP address and hostname

When creating a network, IPv4 address allocation is enabled by default, it
can be disabled using `--ipv4=false`. IPv6 address allocation can be enabled
using `--ipv6`.

```console
$ docker network create --ipv6 --ipv4=false v6net
```

By default, the container gets an IP address for every Docker network it attaches to.
A container receives an IP address out of the IP subnet of the network.
The Docker daemon performs dynamic subnetting and IP address allocation for containers.
Each network also has a default subnet mask and gateway.

You can connect a running container to multiple networks,
either by passing the `--network` flag multiple times when creating the container,
or using the `docker network connect` command for already running containers.
In both cases, you can use the `--ip` or `--ip6` flags to specify the container's IP address on that particular network.

In the same way, a container's hostname defaults to be the container's ID in Docker.
You can override the hostname using `--hostname`.
When connecting to an existing network using `docker network connect`,
you can use the `--alias` flag to specify an additional network alias for the container on that network.

## DNS services

Containers use the same DNS servers as the host by default, but you can
override this with `--dns`.

By default, containers inherit the DNS settings as defined in the
`/etc/resolv.conf` configuration file.
Containers that attach to the default `bridge` network receive a copy of this file.
Containers that attach to a
[custom network](tutorials/standalone.md#use-user-defined-bridge-networks)
use Docker's embedded DNS server.
The embedded DNS server forwards external DNS lookups to the DNS servers configured on the host.

You can configure DNS resolution on a per-container basis, using flags for the
`docker run` or `docker create` command used to start the container.
The following table describes the available `docker run` flags related to DNS
configuration.

| Flag           | Description                                                                                                                                                                                                                                           |
| -------------- |-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `--dns`        | The IP address of a DNS server. To specify multiple DNS servers, use multiple `--dns` flags. DNS requests will be forwarded from the container's network namespace so, for example, `--dns=127.0.0.1` refers to the container's own loopback address. |
| `--dns-search` | A DNS search domain to search non-fully qualified hostnames. To specify multiple DNS search prefixes, use multiple `--dns-search` flags.                                                                                                              |
| `--dns-opt`    | A key-value pair representing a DNS option and its value. See your operating system's documentation for `resolv.conf` for valid options.                                                                                                              |
| `--hostname`   | The hostname a container uses for itself. Defaults to the container's ID if not specified.                                                                                                                                                            |

### Custom hosts

Your container will have lines in `/etc/hosts` which define the hostname of the
container itself, as well as `localhost` and a few other common things. Custom
hosts, defined in `/etc/hosts` on the host machine, aren't inherited by
containers. To pass additional hosts into a container, refer to [add entries to
container hosts file](/reference/cli/docker/container/run.md#add-host) in the
`docker run` reference documentation.

## Container networks

In addition to user-defined networks, you can attach a container to another
container's networking stack directly, using the `--network
container:<name|id>` flag format.

The following flags aren't supported for containers using the `container:`
networking mode:

- `--add-host`
- `--hostname`
- `--dns`
- `--dns-search`
- `--dns-option`
- `--mac-address`
- `--publish`
- `--publish-all`
- `--expose`

The following example runs a Redis container, with Redis binding to
`localhost`, then running the `redis-cli` command and connecting to the Redis
server over the `localhost` interface.

```console
$ docker run -d --name redis example/redis --bind 127.0.0.1
$ docker run --rm -it --network container:redis example/redis-cli -h 127.0.0.1
```
