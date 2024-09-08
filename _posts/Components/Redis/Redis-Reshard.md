---
title: Redis-Reshard
date: 2020-05-26 23:49:25
tags: [Redis] 
---

//todo redis reshard 详情

# Redis 集群Reshrd操作 #

## 交互式Reshard ##

```shell
redis-cli --cluster reshard 127.0.0.1:6379 -a enter_your_password

Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
>>> Performing Cluster Check (using node 127.0.0.1:6379)
M: 2ff327433b2eeb5fc0419377896701271f82589e 127.0.0.1:6379
   slots:[0-5961],[10923-11421] (6461 slots) master
   1 additional replica(s)
S: dd6d82dfae6f51c16a7d91695501b08f4227a4d0 10.10.17.63:6380
   slots: (0 slots) slave
   replicates a2c8731d7639ec26b10fc023ae9ec429658b08a3
M: a2c8731d7639ec26b10fc023ae9ec429658b08a3 10.10.17.64:6379
   slots:[5962-10922] (4961 slots) master
   1 additional replica(s)
S: 15252a5e3bc6b6d4e215694a1b853dac477da80d 10.10.17.63:6381
   slots: (0 slots) slave
   replicates a11b3cf812034b2a5e7afb6b1d887b9f347e3938
S: 48f0c83ec88c27660ba7d0e6b3b1fa072cb9c987 10.10.17.64:6380
   slots: (0 slots) slave
   replicates a11b3cf812034b2a5e7afb6b1d887b9f347e3938
S: f0db649d480b4a2f0a0cdc41016836067abbd67b 10.10.17.65:6380
   slots: (0 slots) slave
   replicates 2ff327433b2eeb5fc0419377896701271f82589e
M: a11b3cf812034b2a5e7afb6b1d887b9f347e3938 10.10.17.65:6379
   slots:[11422-16383] (4962 slots) master
   2 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
How many slots do you want to move (from 1 to 16384)? 1000

lease enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1:all

```

## 自动化Reshard ##

```shell
./redis-cli --cluster reshard <host>:<port> --cluster-from <node-id> --cluster-to <node-id> --cluster-slots <number of slots> --cluster-yes -a <redis_cluster_passwprd>

./redis-cli --cluster reshard 10.10.17.63:6379 --cluster-from all --cluster-to 2ff327433b2eeb5fc0419377896701271f82589e --cluster-slots 1000 --cluster-yes -a enter_your_password

```

# reference #

https://muyinchen.github.io/2016/12/16/Redis%20%E9%9B%86%E7%BE%A4%E6%93%8D%E4%BD%9C/

