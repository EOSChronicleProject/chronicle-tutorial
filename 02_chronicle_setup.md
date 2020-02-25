# Chapter 2. Chronicle setup

## Introduction

[Chronicle](https://github.com/EOSChronicleProject/eos-chronicle) is
an open-source software for processing the output of state history
plugin in `nodeos`. Its development was initially sponsored by 15 EOS
teams, and later additional features were sponsored by various
projects and Telos WPS proposals.

A Chronicle daemon (`chronicle-receiver`) opens two outgoing websocket
connections:

* one to the state history endpoint in nodeos, to request the history
  data;

* one to the consumer process which will receive a stream of JSON
  objects.

If one of these connections fails, the Chronicle process stops and a
lauching tool, such as `systemd`, can restart it again.


## Server requirements

Chronicle keeps its state in a file-mapped shared memory, similar to
the state data of `nodeos`. The requirements for the storage are very
moderate, and this data can easily reside on an HDD array.

RAM requirerements are quite moderate: typically `chronicle-receiver`
occupies a few hundred megabytes in memory. Only when you start
`nodeos` from snapshot, Chronicle requires more RAM while processing
the first block of the snapshot. For example, starting a process for
EOS mainnet may require as much as 12GB RAM and will take more than
half an hour. Once it starts processing blocks, you can stop and
restart it, and RAM consumption will be around 500MB.

While processing historical data, `chronicle-receiver` is typically
utilizing one CPU core at 100%. After catching up, the process is
occupying around 10-20% of CPU core.

Network requirements are very simple: `chronicle-receiver` needs to
establish two outgoing TCP connections, toward nodeos and the consumer
process. You need to plan the network accordingly. Both nodeos and
consumer may reside on different hosts, and even remote locations. The
data protocol does not require quick interactive response, so
round-trip delay of few hundred milliseconds is still acceptable.

Chronicle uses libraries that require a specific version of C++ Boost
library and a GCC compiler version that is not delivered with Ubuntu
18.04. These versions are available in Ubuntu 18.10, so the easiest
way to set up Chronicle is to launch an Ubuntu 18.10 container on your
host. Further examples illustrate it with an LXC container, and there
are also Dicker files supported by community.

The Chronicle page on Github also describes a process of compiling
required components on Ubuntu 18.04 if this OS version is required.


## Consumer process

You need to prepare a consumer process. It may also be several
consumer processes and several Chronicle instances. A consumer prcoess
is opening a websocket service and waits for Chronicle to connect and
start sending data. It should also periodically acknowledge block
numbers that it has processed. Sometimes there's a fork event, and
consumer needs to roll its data back to the forked block.

Details of the data exchange are documented on
[Chronicle](https://github.com/EOSChronicleProject/eos-chronicle)
page.

The [`chronicle-consumer
NPM`](https://www.npmjs.com/package/chronicle-consumer) is designed to
simplify the dApp developer's life. It screens out the Chronicle data
protocol and provides convenient event notifications for the
application service to react on.


There is a [repository with
examples](https://github.com/EOSChronicleProject/chronicle-consumer-npm-examples)
illustrating how to work with the module, using several live
applications on Telos network.

If you don't have a working consumer process and you need to have an
up-to-date Chronicle process that is synchronized with the blockchain,
you can start it with `mode = scan-noexport`, so that it doesn't try
to connect to a consumer.


## Setting up Chronicle

Installation procedure for Ubuntu 18.10 and 18.04 is described on
Chronicle page in detail.

The GitHub repository comes with a systemd unit file that is suitable
for most installations. It allows setting up as many instances as
required, with the directory layout described below. INST stands fot
an instance name that you would like to assign, such as "telos", or
"telos-payments", according to your application needs.

* `/srv/INST/chronicle-config`: configuration directory. Only
  `config.ini` is expected in it. You may add `logging.json` similar
  to that for nodeos, but it's not really necessary because the output
  is not too verbose.

* `/srv/INST/chronicle-data`: Chronicle state data directory. It
  contains a shared memory file, similar to state file of
  `nodeos`. The data file is sparce: normally it's 1GB long, but only
  part of it is occupied. The file contains copies of all ABI
  revisions for all smart contracts, plus some supplimentary state
  tracking data.


The systemd unit file will start `chronicle-receiver` with only two
options, `--config-dir` and `--data-dir`. The rest of configuration is
expected to be present in `config.ini`. But when you start Chronicle
for the first time, you will likely need additional options, so you
can start the process manually. The options in command line are
overriding those in config file.

The data directory can be copied from one Chronicle instance to
another, but `chronicle-receiver` needs to be stopped before copying.

The on-disk format is dependent on compiler and libraries versions, so
copying the data to another server is only making sense if the other
server has the same OS version and had the same Chronicle installation
procedure.

The data file contains copies of ABI for every smart contract, and
their revisions over time. The ABI are required for successful
decoding of binary state history data into JSON.

There are two ways to get ABI for every smart contract into Chronicle
data:

* scan from genesis block to current head: it takes time, and it needs
  a full state history archive accessible by `nodeos`. In
  `scan-noexport` mode this may take from few hours to few days.

* start `nodeos` from a portable snapshot, mark the first block number
  as described in [Chapter 1](01_nodeos_server_setup.md), and start
  Chronicle with `--start-block` option. The state history of such a
  node will contain all RAM contents in the first block. All this data
  is sent as a single response chunk in state history websocket, so
  Chronicle will have to process all this data in a single step. It
  means it needs as much server RAM as it's needed to contain all the
  table deltas from the beginning of the archive. For example,
  starting from an EOS snapshot will require 12GB RAM and about an
  hour of processing. Telos snapshot would require about 20 minutes.


If `nodeos` and Chronicle are starting from a snapshot, there are
several possibilities how to handle the consumer:

* If consumer process needs all table deltas for specific contracts
  that existed before the snapshot, it needs to start and listen to
  Chronicle from the very beginning. As the first block in the
  snapshot has all table contents, your consumer will receive a
  complete copy of contract RAM state in forms of deltas, one per row.

* If you only need to scan the history from a specific block past the
  start of the snapshot, you can start Chronicle with
  `--mode=scan-noexport`, `--start-block=X`, and `--end-block=Y`
  options. Then Chronicle process will stop when it reaches che
  specified end block.
  

## Installatoin example









