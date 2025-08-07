---
title: Docker with nftables
weight: 10
description: How Docker works with nftables
keywords: network, nftables, firewall
---

> [!WARNING]
>
> Support for nftables is experimental, configuration options, behavior and
> implementation may all change in future releases.
> The rules for overlay networks have not yet been migrated from iptables.
> So, nftables cannot be enabled when the daemon has Swarm enabled.

To use nftables instead of iptables, use Docker Engine option
`--firewall-backend=nftables` on its command line, or `"firewall-backend": true`
in its configuration file. See [migrating from iptables to nftables](#migrating-from-iptables-to-nftables).

Docker creates nftables rules in the host's network namespace for bridge
networks. For bridge and other network types, nftables rules for DNS are
also created in the container's network namespace.

Creation of nftables rules can be disabled using daemon options `iptables`
and `ip6tables`, see [Prevent Docker from manipulating firewall rules](packet-filtering-firewalls.md#prevent-docker-from-manipulating-firewall-rules).
But, this option is not appropriate for most users, it is likely to break
container networking.

### Docker's nftables tables

For bridge networks, Docker creates two tables, `ip docker-bridges` and
`ip6 docker-bridges`.

Each table contains a number of [base chains](https://wiki.nftables.org/wiki-nftables/index.php/Configuring_chains#Adding_base_chains),
and further chains are added for each bridge network.

### User-defined nftables rules

Do not modify Docker's tables directly as the modifications are likely to
be lost, Docker expects to have full ownership of its tables.

> [!NOTE]
> 
> Because iptables has a fixed set of chains, equivalent to nftables base
> chains, all rules must be included in those chains. The `DOCKER-USER` chain
> is supplied as a way to include rules in the `filter` table's `FORWARD`
> chain, to run before Docker's rules.
> In Docker's nftables implementation, there is no `DOCKER-USER` chain.
> Instead, rules can be added in a separate table with base chains that
> have the same types and hook points as Docker's base chains. [Base chain
> priority](https://wiki.nftables.org/wiki-nftables/index.php/Configuring_chains#Base_chain_priority)
> can be used to tell nftables which order to call the chains in.

Docker uses well known [priority values](https://wiki.nftables.org/wiki-nftables/index.php/Netfilter_hooks#Priority_within_hook)
for each of its base chains.

### Restrict external connections to containers

By default, any remote host can connect to ports published to the Docker
host's external addresses.

To allow only a specific IP or network to access the containers, create a
table with a base chain that has a drop rule. For example, the
following table drops packets from all IP addresses except `192.0.2.2`:

```console
table ip my-table {
	chain my-filter-forward {
		type filter hook forward priority filter; policy accept;
		iifname ext_if ip saddr 192.0.2.2 counter
	}
}
```

You will need to change `ext_if` to your host's external interface name.

You could instead allow connections from a source subnet. The following
table only allows access from the subnet `192.0.2.0/24`:

```console
table ip my-table {
	chain my-filter-forward {
		type filter hook forward priority filter; policy accept;
		iifname ext_if ip saddr 192.0.2.0/24 counter
	}
}
```

`nftables` is complicated. There is a lot more information at the
[nftables wiki](https://wiki.nftables.org/wiki-nftables/index.php/Main_Page).

## Migrating from iptables to nftables

If the Docker daemon has been running with the iptables firewall backend,
restarting it with the nftables backend will delete most of Docker's iptables
chains and rules, and re-create them using nftables.

You may need to manually update the iptables `FORWARD` policy if it has
been set to `DROP` by Docker with iptables, or as part of your host's
firewall configuration. See [IP forwarding](#ip-forwarding) and [FORWARD
policy in iptables](#forward-policy-in-iptables).

If you have rules in the `DOCKER-USER` chain, see [Migrating `DOCKER-USER`](#migrating-docker-user).

### IP forwarding

When running with iptables, depending on network and daemon configuration,
Docker may enable IPv4 and IPv6 forwarding on the host. This enables port
publishing, communication between bridge networks, and direct routing from
outside the host to containers in bridge networks.

When it enables forwarding, it sets the default policy of the iptables `FORWARD`
chain to `DROP`. This prevents the host from forwarding IP packets between
non-Docker interfaces, including the host's external interfaces and internal
interfaces created by other virtualization software. _Option `ip-forward-no-drop`
tells the daemon not to set that default policy._

With nftables, there is no single `FORWARD` chain. Docker has its own table
with a `FORWARD` chain with rules to accept forwarding related to its own
interfaces. But, "accept" in nftables is not final, packets accepted by one
base chain still get processed by other base chains that have the same type
and hook. So, if it has policy `DROP`, the iptables chain will drop packets
accepted by Docker's chain.

TODO - example rules for blocking forwarding

TODO - list sysctls, say how to set them

TODO - figure out what to say about firewalld's forwarding policies / zones

### FORWARD policy in iptables

An iptables chain with `FORWARD` policy `DROP` will drop packets that have
been accepted by Docker's nftables rules, because the packet will be processed
by iptables chains as well as Docker's nftables chains.

Port publishing will not work unless the `DROP` policy is removed, or additional
iptables rules are added to the iptables `FORWARD` chain to accept Docker-related
traffic.

To check the current iptables `FORWARD` policy, use:

```console
$ iptables -L FORWARD
Chain FORWARD (policy DROP)
target     prot opt source               destination
$ ip6tables -L FORWARD
Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
```

To set the iptables policies to `ACCEPT` for IPv4 and IPv6:

```console
$ iptables -P FORWARD ACCEPT
$ ip6tables -P FORWARD ACCEPT
```

### Migrating `DOCKER-USER`

With firewall backend "iptables", rules added to the iptables `DOCKER-USER`
are processed before Docker's rules in the filter table's `FORWARD` chain.

Most rules in the `DOCKER-USER` will continue to work. For example, if a
packet is dropped, it will be dropped before or after the nftables rules in
Docker's `filter-FORWARD` chain.

> [!NOTE]
>
> When starting the daemon with nftables after running with iptables, Docker
> will not remove the jump from the `FORWARD` chain to `DOCKER-USER`. So,
> rules created in `DOCKER-USER` will continue to run.
>
> But, when starting with nftables, the daemon will not add an iptables
> jump from `FORWARD` to `DOCKER-USER`. So, to preserve`DOCKER-USER` rules
> without migrating them to nftables, you must add that jump when setting
> up the rules in `DOCKER-USER`.

#### Migrating ACCEPT rules

In nftables, an "accept" rule is not final. It terminates processing
for its base chain, but the accepted packet will still be processed by
other base chains, which may drop it.
So, if you're using `DOCKER-USER` to override Docker's "drop" rules, for
example to allow traffic into a container port without publishing that port,
the iptables `ACCEPT` rule will not affect Docker's nftables `drop`.

To override Docker's `drop` rule, you must use a firewall mark. Select a
mark not already in use on your host, and use Docker Engine option
`--bridge-accept-fwmark`.

For example, `--bridge-accept-fwmark=1` tells the daemon to accept any
packet with an `fwmark` value of `1`. Optionally, you can supply a mask
to match specific bits in the mark, `--bridge-accept-fwmark=0x1/0x3`.

Then, instead of accepting the packet in `DOCKER-USER`, add the fwmark
you have chosen and Docker will not drop it.

TODO - example rules

#### Replacing `DOCKER-USER` with an nftables table

Because nftables doesn't have pre-defined chains, to replace the `DOCKER-USER`
chain you can create your own table and add chains and rules to it.

The `DOCKER-USER` chain has type `filter` and hook `forward`, so it can
only have rules in the filter forward chain. The base chains in your
table can have any `type` or `hook`. If your rules need to run before
Docker's rules, give the base chains a lower `priority` number than
Docker's chain. Or, a higher priority to make sure they run after Docker's
rules.

Docker's base chains use the priority values defined at
[priority values](https://wiki.nftables.org/wiki-nftables/index.php/Netfilter_hooks#Priority_within_hook)

TODO - example rules