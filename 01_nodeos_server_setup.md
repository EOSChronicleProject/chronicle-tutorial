# Chapter 1. Setting up a nodeos server

[`nodeos`](https://developers.eos.io/manuals/eos/latest/nodeos/index)
is the core software of an EOSIO blockchain. It is a daemon that
performs all functions of a blockchain node: synchronizing with other
nodes via p2p protocol, providing HTTP API for client software, and
optionally signing blocks if it belongs to a block producer.

Further in this guide, we're only considerinhg non-producing nodes.

`nodeos` is available in source code and binary packages at [EOSIO
GitHub repository](https://github.com/EOSIO/eos). The binary packages
are available in Releases section. Supported platforms are Ubuntu
16.04 and 18.04, RHEL7, and MacOS. Ubuntu 18.04 is recommended, and
will be used further in this guide.


## Planning a server

A server should be blanned accoringly to the network requirements:
busy networks like EOS and Telos require a baremetal server with
directly attached SSD or NVMe, at least for the state files of
`nodeos`. Some hosting providers allow a combination of HDD and SSD in
the same physical server, and blocks log and state history archive can
be stored on hard disks. A typical server would have at least 500GB
SSD storage, 32GB RAM, and a CPU of 3.5Ghz or faster.

Low-traffic networks and testnets would be functioning fine on virtual
private servers. Typically 8GB RAM and a couple of CPU cores will do
the job.

Many hosting providers are offering dedicated hardware, and here are a
few:

* Leaseweb: customizable server configurations, good coverage around
  the world, monthly billing.

* IONOS: multiple locations in Europe, one location in USA, fixed
  server configurations, minute billing, immediate provisioning and
  ceasing.

* Hetzner: fixed and customizable configurations, two locations in
  Germany and Finland, TODO billing

When planning a server installation, you have a number of options to
select from. Often the selection is determined by system
adinistrator's existing habits and preferences.


## Filesystem

ZFS is a filesystem developed by Sun Microsystems and used in their
Solaris OS. The filesystem has been ported onto Linux kernel, and is
available out of the box in Ubuntu 18.04

Other filesystems of choise would be ext4 and XFS. They are stable and
performing well, although ZFS offers a number of unique features to
benefit from:

* Fast and cheap snapshots. It takes a fraction of a second to create
  a new snapshot of a ZFS filesystem, and the snapshot is very
  economical for nodeos data because most of it is append-only (except
  for state file hich is randomly written). The snapshot content can
  later be copied to spme other media or remote location without
  interrupting the service.

* Fast lz4 compression. Even on fast NVMe, the compression does not
  add any visible delays to disk operation, and allows saving up to
  30% in blocks log archive and up to 50% in state file. Also it
  reduces the storage I/O traffic, which allows gaining in
  performance.

* easy creation of as many filesystem as needed, and each filesystem
  would have its own mounting poiint, record size, caching policy, and
  compression settings. This allows fine-tuning of the storage
  according to applicatuion needs.


LVM may or may not be used, and it's up to the system administrator to
define the standard policy. Also some hosting providers, such as
IONOS, are preconfiguring the servers with LVM so it's easier to use
it than to remove it.


## Containers

The system administrator needs to select a strategy for placing nodeos
processes in a server. Typically a server would run multiple nodeos
processes for different purposes, and likely for different networks.

* no containers: nodeos would run directly in the main space of the
  host. the benefit is easier maintenance, and in some cases, better
  performance. The drawback is that the EOSIO binary package installs
  the binaries in one common location, and if multiple nodeos
  processes are running on the server, they all need to be upgraded
  simultaneously.

* LXC containers: most Linux distributions offer LXC containers
  functionality. Debian supposes that you know how to set up a network
  for the containers, wile Ubuntu preconfigures an internal bridge
  with addresses in 10.0.3.0/24 range and a DHCP service iin it. The
  addressing is easy to change, and also DHCP service can be
  configured with static address assignments for your containers. When
  created, a container is a bare minimum Linux OS, and you need to
  install additional packages as needed (even syslog and ping are not
  present from the beginning). The containers are easy to maintain,
  and one of benefits is that you can select a different Lunux
  distribution or release than what the host is running.

* Docker containers: they use the same underlying technology as LXC,
  but there's a number of additional features, such as automatic
  deployment. The author does not have much experience with Docker,
  although many people prefer using it.`

* KVM, Xen, VmWare virtualization: there is a certain overhead in CPU
  operation, and typically full virtualization is not used in EOSIO
  environment. However, it might be usable, especially if it's already
  a part of standard IT processes.

Further in this guide, LXC containers are used.


## Network

Network layout and security need to be planned carefully.

In most hosting provider environments, the baremetal machines are
directly facing the public internet without any firewalls in front of
them. Some providers offer a firewall in front of a server, optionally
or mandatory.

Some hosting providers are offering floating IP addresses: an IP
address can be dynamically assigned to one of your servers and
switched over to your other server, based on some condition
monitoring.

Some hosting providers are offering load balancers in front of the
servers. A load balancer is usually monitoring the connectivity to the
backend servers and stops sending requests to a server that stops
responding, or some custom monitoring condition is met. They also
normally allow you to pause the traffic to one of servers and take ot
out of service for maintenance and upgrades. Normally such a load
balancer is only allowing backend servers belonghing to the same
service provider and the same location.

There are several DNS hosting providers offering geo-aware service:
the query response depends on the region where the reuest was sent
from. This allows you to set up several servers across the world,
answering to the same DNS name. One of suchh providers is [DNS Masde
Easy](https://dnsmadeeasy.com/).

A nodeos process is typically listens on 2 or 3 TCP sockets: the p2p
endpoint, the HTTP API, and optionally state history plugin websocket.

Also nodeos is typically establishing outgoing TCP connections to its
p2p peers as specified in its configuration.

Further in this guide, we utilize the state history plugin, so the
nodeos process will listen to 3 TCP sockets. You may typically decide
to configure corresponding binding addresses as `0.0.0.0` or
`127.0.0.1`, depending on how you plan the connectivity.

Chronicle process will need to connect to the state history endpoint,
so the binding address and port should be reachable for it.


## Telos node example

Telos uses vanilla EOSIO software, so binary packages from EOSIO
GitHub reporitory will work. Few other networks require their own
versions of EOSIO (such as BIS and WAX).

The [tools.eosmetal.io](https://tools.eosmetal.io/snapshots) website
provides fresh portable snapshots for Telos mainnet and testnet, as
well as a [list of p2p
endpoints](https://tools.eosmetal.io/nodestatus/telos).

Typically you download the most recent snapshot according to your
requirements: if your application needs to collect the historical data
starting from some point in time, you download a snapshot from the day
prior to that. Otherwise, you download the latest snapshot.




