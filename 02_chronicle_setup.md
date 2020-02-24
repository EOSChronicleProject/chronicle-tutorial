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





