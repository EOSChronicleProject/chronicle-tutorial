# Chronicle tutorial

This tutorial aims giving an EOSIO dApp developer clear guidelines how
to set up a blockchain server and start collecting data for an
application. This includes historical and real-time data, such as
transaction traces and contract table contents.

Examples in this tutorial are made for telos mainnet. Users of other
EOSIO blockchains need to adapt their setup accordingly.

This development is sponsored by [Telos
blockchain](https://www.telos.net/) community as part of [Worker
Proposal #95](https://chainspector.io/governance/worker-proposals/95).

[Chronicle](https://github.com/EOSChronicleProject/eos-chronicle) is a
software package for receiving and decoding the data flow that is
exported by `state_history_plugin` of `nodeos`, the blockchain node
daemon [developed by Block One](https://developers.eos.io/).

[`chronicle-consumer
NPM`](https://www.npmjs.com/package/chronicle-consumer) is a Node.js
module for a server-side application that needs to receive blockchain
updates from Chronicle. It comes along with a number of
[examples](https://github.com/EOSChronicleProject/chronicle-consumer-npm-examples)
that illustrate how to collect data for some dApps on Telos.

## Contents

* [Chapter 1. Setting up a nodeos server](01_nodeos_server_setup.md)

* [Chapter 2. Setting up Chronicle](02_chronicle_setup.md)





## Copyright and License

Copyright 2020 cc32d9@gmail.com

This work is licensed under a Creative Commons Attribution 4.0
International License.

http://creativecommons.org/licenses/by/4.0/
