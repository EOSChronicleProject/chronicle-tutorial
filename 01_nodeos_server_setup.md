# Chapter 1. Setting up a nodeos server

`nodeos` is the core daemon in the [Leap software
package](https://github.com/AntelopeIO/leap) and is the base element of
an [Antelope](https://antelope.io/) (former EOSIO) blockchain. It is a
daemon that performs all functions of a blockchain node: synchronizing
with other nodes via p2p protocol, providing HTTP API for client
software, and optionally signing blocks if it belongs to a block
producer.

Further in this guide, we're only considerinhg non-producing nodes. A
detailed explanation of node roles and functions is available in the
[Antelope Developers's
Handbook](https://cc32d9.gitbook.io/antelope-smart-contract-developers-handbook/).

`nodeos` is available in source code and binary packages at the [Leap
GitHub
repository](https://github.com/AntelopeIO/leap/releases). Supported
platforms are Ubuntu 18.04, 20.04, and 22.04.

## Planning a server

A server should be planned in accordance with the network
requirements: busy networks like EOS, Telos and WAX require a
baremetal server with directly attached SSD or NVMe. Some hosting
providers allow a combination of HDD and SSD in the same physical
server, and blocks log and state history archive can be stored on hard
disks.

As of beginning of 2023, the WAX blockchain has the largest state
among public networks, occupying about 100GB. Also the intensity of
transactions is so high that in default state mappimg mode (state
mapped to a file on NVS storage) the speed of storage is sometimes not
high enough, and the drives are worn off relatively quickly. An
enterprise grade NVME drive could be desroyed within a few months.

The current recommended setup for a WAX node is utilizing a `tmpfs`
partition for the state directory. The tmpfs filesystem maps its
content to the computer RAM and swap space. The Linux kernel is very
efficient in maintaining the frequently updated pages in RAM and
pushing unused pages to the swap.

Another aspect that needs to be taken into account is that the state
history plugin needs additional RAM when a node starts from a
snapshot: it writes all the contents of state memory into the first
block of its table deltas file, and it needs to prepare the image in
the memory before writing it to the disk.

A recommended setup for a WAX state history server is an Intel Xeon or
AMD CPU, 128 GB RAM with ECC, at least 256GB swap space on SSD or
NVME, and at least a terabyte of storage for state history and blocks
log. ZFS filesystem is recommended for the logs storage, as its
built-in lz4 compression is reducing the space occupied by the blocks
log. The state history archives are already compressed by nodeos, so
filesystem compression does not add any benefits for them.

An EOS or Telos server would be alright with 32GB RAM, although 64GB
would be recommended. Also the big WAX server with 128GB RAM and big
swap can easily accommodate additional EOS and Telos nodes. They just
need to start from snapshots one after the other.

Low-traffic networks and testnets would be functioning fine on virtual
private servers. Typically 8GB RAM and a couple of CPU cores will do
the job.

Many hosting providers are offering dedicated hardware, and here are a
few:

* Leaseweb: customizable server configurations, good coverage around
  the world, monthly billing.

* IONOS: multiple locations in Europe, one location in USA, fixed
  server configurations, minute billing, immediate provisioning and
  ceasing. The OS installer does not allow any customizig in the
  storage layout, so it's not convenient for long-term blockhain
  nodes. Also some network stability issues have been known in the
  past.

* Hetzner: fixed and customizable configurations, two locations in
  Germany and Finland. Some server types can be provisioned within
  minutes.

* Vultr: baremetal servers available in many locations around the
  globe. Only a few standard configuations are available, and storage
  partitioning is also limited in options.

* OVH: a large variety of baremetal servers around the globe,
  customizable hardware configuration and storage partitioning.

A full archive of blocks log and state history may become quite
expensive to maintain, as it counts in terabytes for most active
networks. In most tasks which are related to Chronicle, you would only
be interested in newest blockchain history, and the old history can
periodically discarded. This allows saving on hosting costs.


## ZFS Filesystem

ZFS is a filesystem developed by Sun Microsystems and used in their
Solaris OS. The filesystem has been ported onto Linux kernel, and is
available out of the box in Ubuntu 18.04

Other filesystems of choise would be ext4 and XFS. They are stable and
performing well, although ZFS offers a number of unique features to
benefit from:

* Fast and cheap snapshots. It takes a fraction of a second to create
  a new snapshot of a ZFS filesystem, and the snapshot is very
  economical for nodeos data because most of it is append-only (except
  for state file which is randomly written). The snapshot content can
  later be copied to some other media or remote location without
  interrupting the service.

* Fast lz4 compression. Even on fast NVMe, the compression does not
  add any visible delays to disk operation, and allows saving up to
  30% in blocks log archive and up to 50% in state file. Also it
  reduces the storage I/O traffic, which allows gaining in
  performance.

* easy creation of as many filesystems as needed, and each filesystem
  would have its own mounting point, record size, caching policy, and
  compression settings. This allows fine-tuning of the storage
  according to applicatuion needs.


## tmpfs Filesystem

`tmpfs` is a filesystem mapped to the computer memory. If the computer
reboots, the contents of a tmpfs system are lost. When you create a
tmpfs partition, you specify its allocation size. This amount of RAM
is not allocated immediately, but the RAM allocation grows as more
content is written. Eventually, if the written content occupies all
the filesystem, the corresponding amount of computer RAM gets
allocated.

If a `nodeos` state is mapped to a file in a `tmpfs` filesystem, the
Linux kernel is doing two smart things: there's no double allocation
of memory. When node accesses its file-mapped shared memory, the
process reads or writes directly in the memory area where tmpfs places
it. Also, the kernel is using the swap storage in a smart way, keeping
the most actively used pages in the server RAM.

If the state is mapped to a `tmpfs` partition, you may notice that a
significant part of swap storage is being used, but the storage I/O
rate is very moderate.



## Containers

The system administrator needs to select a strategy for placing nodeos
processes on a server. Typically a server would run multiple nodeos
processes for different purposes, and likely for different networks.

* no containers: nodeos would run directly in the main space of the
  host. The benefit is in easier maintenance, and in some cases,
  better performance. The drawback is that the Antelope binary package
  installs the binaries in one common location, and if multiple nodeos
  processes are running on the server, they all need to be upgraded
  simultaneously. But using `dpkg` command, one may unpack the package
  contents in a non-standard location.

* LXC containers: most Linux distributions offer LXC containers
  functionality. Debian supposes that you know how to set up a network
  for the containers, wile Ubuntu preconfigures an internal bridge
  with addresses in 10.0.3.0/24 range and a DHCP service in it. The
  addressing is easy to change, and also DHCP service can be
  configured with static address assignments for your containers. When
  created, a container is a bare minimum Linux OS, and you need to
  install additional packages as needed (even syslog and ping are not
  present from the beginning). The containers are easy to maintain,
  and one of benefits is that you can select a different Linux
  distribution or release than what the host is running.

* Docker containers: they use the same underlying technology as LXC,
  but there's a number of additional features, such as automatic
  deployment. The author does not have much experience with Docker,
  although many people prefer using it.

* KVM, Xen, VmWare virtualization: there is a certain overhead in CPU
  operation, and typically full virtualization is not used in Antelope
  environment. However, it might be usable, especially if it's already
  a part of standard IT processes.

Further in this guide, LXC containers are used.


## Network

Network layout and network security need to be planned carefully.

In most hosting provider environments, baremetal machines are directly
facing the public internet without any firewalls in front of
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
normally allow you to pause the traffic to one of servers and take it
out of service for maintenance and upgrades. Normally such a load
balancer is only allowing backend servers belonghing to the same
service provider and the same location.

There are several DNS hosting providers offering geo-aware service:
the query response depends on the region where the reuest was sent
from. This allows you to set up several servers across the world,
answering to the same DNS name. One of such providers is
[DNScaster](https://dnscaster.com/).

A nodeos process listens typically on 2 or 3 TCP sockets: the p2p
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

Telos uses vanilla Antelope software, so binary packages from Antelope
GitHub reporitory will work. Few other networks require their own
customized versions of Antelope (such as WAX and FIO).

The [snapshots service by EOS Nation](https://snapshots.eosnation.io/)
provides fresh portable snapshots for a number of public blockchains,
including Telos mainnet and testnet.

Typically you download the most recent snapshot according to your
requirements: if your application needs to collect the historical data
starting from some point in time, you download a snapshot from the day
prior to that. Otherwise, you download the latest snapshot.

In this example, we have a server with two NVME and two HDD drives and
128GB RAM. The server is enough to serve a Telos, WAX and EOS nodes at
the same time, but this example shows the Telos part only.

A part of NVME drives is occupied by the OS, then we allocate a 128GB
partition for swap space on each drive, and the rest is used by ZFS
"zfast" pool, which is ised for the LXC containers.

The hard drives are used by "zslow" ZFS pool, and here we store the
blocks log and state history.

Bear in mind that a WAX node needs a different nodeos executable,
installed separately.


```
apt-get update && apt-get install -y aptitude git lxc-utils zfsutils-linux \
 netfilter-persistent sysstat ntp

# swap partitions
mkswap /dev/nvme1n1p3
mkswap /dev/nvme0n1p3

cat >>/etc/fstab <<'EOT'
/dev/nvme1n1p3       none            swap    sw
/dev/nvme0n1p3       none            swap    sw
EOT

# the rest of NVME space used for ZFS
zpool create -o ashift=12 zfast mirror /dev/nvme1n1p4 /dev/nvme0n1p4
zfs set atime=off zfast
zfs set compression=lz4 zfast

# the bulk storage for history files
zpool create -o ashift=12 zslow mirror /dev/sda  /dev/sdb
zfs set atime=off zslow
zfs set compression=lz4 zslow

# this tells the LXC DHCP service to read the config file for static
# IP address assignment:

sed -i -e 's,^.*LXC_DHCP_CONFILE,LXC_DHCP_CONFILE,' /etc/default/lxc-net
cat >/etc/lxc/dnsmasq.conf <<'EOT'
EOT
systemctl restart lxc-net



###########   Antelope container   #############

# "eosio" container will use this IP address on internal virtual bridge.
cat >>/etc/lxc/dnsmasq.conf <<'EOT'
dhcp-host=eosio,10.0.3.20
EOT
systemctl restart lxc-net

zfs create -o mountpoint=/var/lib/lxc/eosio -o compression=lz4 zfast/eosio

# Telos node with data in /srv/telos/data

# blocks and snapshots are compressing well, so we enable compression by default
zfs create -o mountpoint=/var/lib/lxc/eosio/rootfs/srv/telos/data \
 -o compression=lz4 zslow/eosio/telos_data

# state history is compressed on data level, so filesystem compression
# will not add any benefits
zfs create -o mountpoint=/var/lib/lxc/eosio/rootfs/srv/telos/data/state-history \
 -o compression=off zslow/eosio/telos_ship

# nodeos state in tmpfs. Note the mount sequence, as we need ZFS to come first
mkdir -p /var/lib/lxc/eosio/rootfs/srv/telos/data/state
cat >>/etc/fstab <<'EOT'
tmpfs   /var/lib/lxc/eosio/rootfs/srv/telos/data/state  tmpfs rw,nodev,nosuid,size=32G,x-systemd.after=zfs-mount.service 0  0
EOT

# mount the tmpfs
mount -a

# restart in order to make sure that ZFS, swap and tmpfs are
# initialized correctly at boot.
reboot

# Ubuntu 22.04 container for the nodes
lxc-create -n eosio -t download -- --dist ubuntu --release jammy --arch amd64

# this will start the container automatically with the host
echo "lxc.start.auto = 1" >> /var/lib/lxc/eosio/config

lxc-start -n eosio
lxc-attach -n eosio

# now we're in a shell within the container. LXC creates the user
# "ubuntu" by default, which is not needed
apt-get update && apt-get install -y git openssh-server net-tools
userdel -rf ubuntu
exit

# here we assume that root on the host has authorized keys in its
# .ssh/ folder, and you want the same keys to access the container via
# local SSH. You may want a different security policy instead.
cp -a /root/.ssh/ /var/lib/lxc/eosio/rootfs/root/
ssh -A root@10.0.3.20


# you need to check for the latest Antelope software release at
#  https://github.com/AntelopeIO/leap/releases

cd /var/local
wget https://github.com/AntelopeIO/leap/releases/download/v3.2.2/leap_3.2.2-ubuntu22.04_amd64.deb
apt install -y ./leap_3.2.2-ubuntu22.04_amd64.deb


# Now we need to create nodeos configuration. There's a number of
# parameters that may vary, depending on your requirements:

# chain-state-db-size-mb needs to be big enough to accomodate for the
# current blockchain state. It should also not exceed the size of
# tmpfs partition.

# http-server-address, p2p-listen-endpoint, and state-history-endpoint
# define the TCP ports that your node would listen to, for HTTP, p2p,
# and state history requests, correspondingly. If you run multiple
# nodes on the same server, they should listen on different TCP ports.

# sync-fetch-span defines how many blocks your node will request from
# p2p peers when resynching with the network.

# p2p peer addresses: a server normally doesn't need more than 5-6
# peers, and they should preferably be geographically close, for
# faster resync. The tools like https://validate.eosnation.io/ provide
# automatically generated lists of peers for multiple networks.

# contracts-console and trace-history-debug-mode: if both are set to
# true, the state history traces will contain the ouput of "print"
# statements within smart contracts. This would occupy more space in
# the state history archive, and it's rarely used.

# p2p-accept-transactions and api-accept-transactions, when set to
# false, will make sure that the node is only processing signed
# blocks, and the CPU is not busy with speculative transactions.

mkdir /srv/telos/etc/

cat >/srv/telos/etc/config.ini <<'EOT'
chain-state-db-size-mb = 30000

wasm-runtime = eos-vm-jit
eos-vm-oc-enable = true

p2p-accept-transactions = false
api-accept-transactions = false

http-server-address = 0.0.0.0:8881
p2p-listen-endpoint = 0.0.0.0:9851

access-control-allow-origin = *
contracts-console = true

plugin = eosio::chain_plugin
plugin = eosio::chain_api_plugin

plugin = eosio::state_history_plugin
trace-history = true
chain-state-history = true
trace-history-debug-mode = true
state-history-endpoint = 0.0.0.0:8081

sync-fetch-span = 200

p2p-peer-address = telosp2p.3dkrender.com:9888
p2p-peer-address = telosqm.3dkrender.com:9888
p2p-peer-address = telosp2p.actifit.io:9876
p2p-peer-address = telos.eu.eosamsterdam.net:9120
p2p-peer-address = telos.p2p.boid.animus.is:5151
p2p-peer-address = telos.p2p.boid.animus.is:5252
p2p-peer-address = p2p.telos.cryptobloks.io:9876
p2p-peer-address = node-telos.eosauthority.com:10311
p2p-peer-address = telosp2p.eos.barcelona:2095
EOT


# By default, nodeos is pretty verbose. This logging configuration
# leaves only errors:

cat >/srv/telos/etc/logging.json <<'EOT'
{
  "includes": [],
  "appenders": [{
      "name": "consoleout",
      "type": "console",
      "args": {
        "stream": "std_out",
        "level_colors": [{
            "level": "debug",
            "color": "green"
          },{
            "level": "warn",
            "color": "brown"
          },{
            "level": "error",
            "color": "red"
          }
        ]
      },
      "enabled": true
    }
  ],
  "loggers": [{
      "name": "default",
      "level": "error",
      "enabled": true,
      "additivity": false,
      "appenders": [
        "consoleout"
      ]
    }
  ]
}
EOT


# Now, you have two options to launch your node:

# 1). starting from genesis will produce the whole history of the
# blockchain.  Synchronization will take days or months, depending on
# the chain and p2p peer proximity. If you plan do do so, leave only
# one or two p2p peers in your configuration, and these peers should
# not be geographically too far. This will help you increase the
# resync speed.

# 2). starting from snapshot. https://snapshots.eosnation.io/ provides
# a good list of daily snapshots. This will allow you to process all
# the state history data starting from the snapshot block. Plus, the
# beginning of state history archive will contain table deltas for all
# tables, so you would get a full copy of state in the history
# archive.

# In both cases, you start nodeos from command line, specifying either
# snapshot or genesis file. Starting from genesis is very fast, while
# starting from snapshot will take some time. As soon as you see the
# node's TCP ports open in "netstat -an | grep LISTEN", it means you
# can stop the process with Ctrl-C or "kill" command, and it's ready
# for launching as a systemd service.

# --disable-replay-opts option in nodeos is required for state history
# plugin to work

# After starting a node, you will likely see many errors related to
# p2p peers rejecting connections. This is normal, as many public
# endpointts are limited in capacity. Your node will still need to
# have a few peers connected for stable operation.

# ## start from genesis. Below is genesis file for Telos mainnet ##

cat >/srv/telos/etc/genesis.json <<'EOT'
{
 "initial_key": "EOS52vfcN43YHHU8Akh7VyfBdnDiMg15dPTELosWG9SR86ssBoU1T",
 "initial_configuration": {
   "max_transaction_delay": 3888000,
   "min_transaction_cpu_usage": 100,
   "net_usage_leeway": 500,
   "context_free_discount_net_usage_den": 100,
   "max_transaction_net_usage": 524288,
   "context_free_discount_net_usage_num": 20,
   "max_transaction_lifetime": 3600,
   "deferred_trx_expiration_window": 600,
   "max_authority_depth": 6,
   "max_transaction_cpu_usage": 5000000,
   "max_block_net_usage": 1048576,
   "target_block_net_usage_pct": 1000,
   "max_generated_transaction_count": 16,
   "max_inline_action_size": 4096,
   "target_block_cpu_usage_pct": 1000,
   "base_per_transaction_net_usage": 12,
   "max_block_cpu_usage": 50000000,
   "max_inline_action_depth": 4
 },
 "initial_timestamp": "2018-12-12T10:29:00.000"
}
EOT

/usr/bin/nodeos --data-dir /srv/telos/data --config-dir /srv/telos/etc \
 --disable-replay-opts --genesis-json=/srv/telos/etc/genesis.json &

# nodeos started in background. Check the TCP ports with
# "netstat -an | grep LISTEN" and wait till nodeos starts listening
# on configured ports. When it is so, execure "kill %1" to stop the
# background job.


# ## start from snapshot ##

cd /var/local
wget https://pub.store.eosnation.io/telos-snapshots/snapshot-2022-11-29-23-telos-v4-0249634428.bin.zst
zstd -d snapshot-2022-11-29-23-telos-v4-0249634428.bin.zst

# start nodeos from the snapthot. See the instructions about netstat above.

/usr/bin/nodeos --data-dir /srv/telos/data --config-dir /srv/telos/etc --disable-replay-opts \
 --snapshot=snapshot-2022-11-29-23-telos-v4-0249634428.bin

# once the TCP ports are visible in netstat output, stop the background p[rocess with "kill %1"

# ## systemd service ##

# Now nodeos is ready to run as a service. Create a systemd unit file
# and activate the service. There are two options of your preference:
# one unit file per nodeos process, or a template unit file using the
# "@" symbol and %i as service identifier. Below example is
# illustrating one unit file per nodeos process:


cat >/etc/systemd/system/telos.service <<'EOT'
[Unit]
Description=Telos
[Service]
Type=simple
ExecStart=/usr/bin/nodeos --data-dir /srv/telos/data --config-dir /srv/telos/etc --disable-replay-opts
TimeoutStartSec=30s
TimeoutStopSec=300s
Restart=no
User=root
Group=daemon
KillMode=control-group
[Install]
WantedBy=multi-user.target
EOT

# reload systemd so that it sees the new unit file
systemctl daemon-reload

# enable automatic start and start the daemon
systemctl enable telos
systemctl start telos

# watch the nodeos log
journalctl -u telos -f

# check the node synchronization status
date; cleos -u http://127.0.0.1:8881 get info


# restart and find the start block number in state history archive:
systemctl restart telos
journalctl -u telos

# the startup log lines should look like:
Feb 22 22:58:03 eosio nodeos[22003]: info  2020-02-22T22:58:03.264 nodeos    state_history_log.hpp:215     open_log             ] trace_history.log has blocks 68997517-75193929
Feb 22 22:58:03 eosio nodeos[22003]: info  2020-02-22T22:58:03.274 nodeos    state_history_log.hpp:215     open_log             ] chain_state_history.log has blocks 68997517-75193929

# take a note of the start block number, you will need it to initialize Chronicle,

# now wait till the node synchronizes
```


## Monitoring tools

If you are using Nagios or Icinga for service monitoring, [eos-nagios-plugins
scripts](https://github.com/cc32d9/eos-nagios-plugins) will be useful
for nodeos health checkup.

Also
[eosio_net_monitor](https://github.com/eos-amsterdam-rnd/eosio_net_monitor)
will be useful to see the status of your p2p peers.








