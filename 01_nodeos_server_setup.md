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
  would have its own mounting point, record size, caching policy, and
  compression settings. This allows fine-tuning of the storage
  according to applicatuion needs.


LVM may or may not be used, and it's up to the system administrator to
define the standard policy. Also some hosting providers, such as
IONOS, are preconfiguring the servers with LVM so it's easier to use
it than to remove it.

`nodeos` requires access to its state shared memory with minimum
possible delay. So, the `state/` folder in its data directory should
reside on SSD or NVMe storage. Other directories, such as `blocks` and
`state-history`, are quite low on I/O performance requirements, and
hard disks are performing well for this job. Also the blocks log and
state history may require signinicant storage space.

In most tasks which are related to Chronicle, you would only be
interested in newest blockchain history, and the old one can
periodically truncated. This allows saving on hosting costs.


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
normally allow you to pause the traffic to one of servers and take ot
out of service for maintenance and upgrades. Normally such a load
balancer is only allowing backend servers belonghing to the same
service provider and the same location.

There are several DNS hosting providers offering geo-aware service:
the query response depends on the region where the reuest was sent
from. This allows you to set up several servers across the world,
answering to the same DNS name. One of suchh providers is [DNS Made
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
versions of EOSIO (such as BOS and WAX).

The [tools.eosmetal.io](https://tools.eosmetal.io/snapshots) website
provides fresh portable snapshots for Telos mainnet and testnet, as
well as a [list of p2p
endpoints](https://tools.eosmetal.io/nodestatus/telos).

Typically you download the most recent snapshot according to your
requirements: if your application needs to collect the historical data
starting from some point in time, you download a snapshot from the day
prior to that. Otherwise, you download the latest snapshot.

In this example, we have one storage volume and create one ZFS pool on
it: "zdata".

If your server has both SSD and HDD arrays, you may create two
separate ZFS pools (call it "zfast" and "zslow", for example), and use
the fast one for nodeos state. The rest can reside on HDD without much
impact on performance.


```
apt-get update && apt-get install -y aptitude git lxc-utils zfsutils-linux \
 netfilter-persistent sysstat ntp

# Example for a dedicated host at IONOS: most of the storage space is
# unallocated and under LVM, so we create a new volume group for all
# available space:

lvcreate -n zdata -l 100%FREE vg00

# create a ZFS pool with compression enabled by default
zpool create zdata /dev/vg00/zdata
zfs set atime=off zdata
zfs set compression=lz4 zdata

# this tells the LXC DHCP service to read the config file for static
# IP address assignment:

sed -i -e 's,^.*LXC_DHCP_CONFILE,LXC_DHCP_CONFILE,' /etc/default/lxc-net
cat >/etc/lxc/dnsmasq.conf <<'EOT'
EOT
systemctl restart lxc-net


###########   EOSIO container   #############

# "eosio" container will use this IP address on internal virtual bridge.
cat >>/etc/lxc/dnsmasq.conf <<'EOT'
dhcp-host=eosio,10.0.3.20
EOT
systemctl restart lxc-net

zfs create -o mountpoint=/var/lib/lxc/eosio -o compression=lz4 zdata/eosio

# Telos node with data in /srv/telos/data

# blocks and snapshots are compressing well, so we enable compression by default
zfs create -o mountpoint=/var/lib/lxc/eosio/rootfs/srv/telos/data \
 -o compression=lz4 zdata/eosio/telos_data

# state history is compressed on data level, so filesystem compression
# will not add any benefits
zfs create -o mountpoint=/var/lib/lxc/eosio/rootfs/srv/telos/data/state-history \
 zdata/eosio/telos_ship

# nodeos state is read and written randomly in 4k pages, so the record size is
# adjusted accordingly. The data is virtual memory mapped to a file, so no need
# to cache its content on ZFS level.
zfs create -o mountpoint=/var/lib/lxc/eosio/rootfs/srv/telos/data/state \
 -o recordsize=4k -o primarycache=metadata -o compression=lz4 zdata/eosio/telos_state


# this command sometimes failes because PGP key server is too busy. In
# this case, you need to delete /var/lib/lxc/eosio/config and try again
lxc-create -n eosio -t download -- --dist ubuntu --release bionic --arch amd64

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


# you need to check for the latest EOSIO software release at
#  https://github.com/EOSIO/eos/releases

cd /var/local
wget https://github.com/EOSIO/eos/releases/download/v2.0.3/eosio_2.0.3-1-ubuntu-18.04_amd64.deb
apt install -y ./eosio_2.0.3-1-ubuntu-18.04_amd64.deb


# Now we need to create nodeos configuration. There's a number of
# parameters that may vary, depending on your requirements:

# http-server-address, p2p-listen-endpoint, and state-history-endpoint
# define the TCP ports that your node would listen to, for HTTP, p2p,
# and state history requests, correspondingly.

# sync-fetch-span defines how many blocks your node will request from
# p2p peers when resynching with the network.

# p2p peer addresses: they EOSIO 2.0 is handling the peer traffic much
# more efficiently than 1.8, so a server may easily handle 20-30
# peers. You need to retrieve a fresh list of p2p endpoints. For
# example, https://tools.eosmetal.io/nodestatus/telos is a good source
# of them.

# contracts-console and trace-history-debug-mode: if both are set to
# true, the state history traces will contain the ouput of "print"
# statements within smart contracts.

# read-mode: if set to "read-only", the node will only process blocks
# signed by block producers. It will not process any speculative
# transactions and will not accept push_transaction API call. This
# parameter also allows you to set the node in irreversible-only mode,
# which is needed only rarely.

mkdir /srv/telos/etc/

cat >/srv/telos/etc/config.ini <<'EOT'
chain-state-db-size-mb = 65536
reversible-blocks-db-size-mb = 2048

wasm-runtime = eos-vm-jit
eos-vm-oc-enable = true
read-mode = read-only

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

p2p-peer-address = peer.telos.alohaeos.com:9876
p2p-peer-address = p2p.telosvoyager.io:9876
p2p-peer-address = mainnet.get-scatter.com:9876
p2p-peer-address = p2p.telos.africa:9877
p2p-peer-address = p2p.telosuk.io:9876
p2p-peer-address = telosseed.ikuwara.com:9876
p2p-peer-address = seed.tlos.goodblock.io:9876
p2p-peer-address = telos.eossweden.eu:8012
p2p-peer-address = p2p2.telos.telosgreen.com:9877
p2p-peer-address = node2.emea.telosglobal.io:9876
p2p-peer-address = api.telos.kitchen:9876
p2p-peer-address = p2p.telosarabia.net:9876
p2p-peer-address = peer1-telos.eosphere.io:9876
p2p-peer-address = peer2-telos.eosphere.io:9876
p2p-peer-address = telosapi.eosmetal.io:29877
p2p-peer-address = telosp2p.eoscafeblock.com:9120
p2p-peer-address = p2p.theteloscope.io:9876
p2p-peer-address = telosp2p.eos.barcelona:9876
p2p-peer-address = telos.caleos.io:9880
p2p-peer-address = node1.apac.telosglobal.io:9876
p2p-peer-address = p2p-telos-21zephyr.maltablock.org:9876
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
# blockchain.  synchronization will take a few days. If you plan do do
# so, leave only one or two p2p peers in your configuration,. and
# these peers shoudl not be geographically too far. This will help you
# increase the resync speed.

# 2). starting from snapshot. https://tools.eosmetal.io/snapshots
# provides a good list of snapshots for few months into the past. This
# will allow you process all the state history data starting from the
# snapshot block. Plus, the beginning of state history archive will
# contain table deltas for all tables, so you would get a full copy of
# state in the history archive.

# In both cases, you start nodeos from command line, specifying either
# snapshot or genesis file. Starting from genesis is very fast, while
# starting from snapshot will take a few minutes. As soon as you see
# the node's TCP ports open in "netstat -an | grep LISTEN", it means
# you can stop the process with Ctrl-C or "kill" command, and it's
# ready for launching as a systemd service.

# --disable-replay-opts option in nodeos is required for state history
# plugin to work

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

# Find the most recent snapshot at https://tools.eosmetal.io/snapshots
# download and unpack it

cd /var/local
wget https://eosmetal.io/snapshots/telos/2019-09-04-12-00-23-v1.8.2.tar.gz
tar xzvf 2019-09-04-12-00-23-v1.8.2.tar.gz

# start nodeos from snapthos. See the instructions about netstat above.

/usr/bin/nodeos --data-dir /srv/telos/data --config-dir /srv/telos/etc --disable-replay-opts \
 --snapshot=opt/telos/data-dir/snapshots/snapshot-02b8df5ac8e9b622eea6d2f9e0ecbee6b4eb6c62d3de3c7ad621d12930589a67.bin 

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








