---
title: Redis-Ligecycle
date: 2020-05-26 23:59:25
tags: [Redis] 
---

如何安装 redis, 启动 Redis 和 redis 集群设置。

<!--more-->

## Installation ##

There are many ways to install redis on different version system. Using package managment tool like brew, apt-get, yum ans so on is more convenient than other ways. But in this case we would install redis by tarball. 

### Prerequisite to be installed ###

* Install necessary package

yum -y install gcc-c++

> -y: always yes

* Install ruby

1. Download ruby tarball
2. Unarchive ruby package
3. Set access authority(Optional): chmod -R 700 ruby-2.6.3
4. cd redis_install_path/ruby-2.6.3/
5. Set ruby install path: ./configure --prefix=redis_install_path/ruby
6. Compile code: make
7. Compile and install: make install
8. Add ruby command folder to system path: export PATH=redis_install_path/ruby/bin:$PATH

> Tips: If there are two executable files with same name in different folder and their path are both in PATH. The previous ones in PATH will be executed, and the latter will be ignored.

* Install rubygem(gem is a package management tool in ruby) 

1. Download rubygems tarball 
2. Unarchive rubygems packages: unzip rubygems-3.0.4.zip
3. Set access authority(Optional): chmod -R 700 rubygems-3.0.4
4. cd rubygems-3.0.4
5. Install ruby-gems: ruby setup.rb

* Install redis-gem(Install the redis-gem package using gem)

1. Download redis-gems tarball
2. Install: gem install -l redis-4.1.2.gem

### Download, build, and install Redis server ###

* Install redis 

1. Download redis installation package
2. Unarchive package: tar -xvzf redis-stable.tar.gz
3. cd redis-stable
4. Compile: make
5. Install: make install

---

## Start redis instance ##

1. Update redis configuratio file(Refer to redis_config.conf.j2 file)

2. Create the pid, log, configuration folder

3. Start redis by configuration file: 

   > redis-server {{ config_directory }}/{{redis_port}}.conf

```yml
# The following configuration is important
port <REDIS PORT>

# Specify the log file name
logfile "redisLog.log"

# Set pid file path where port is the port where it is running
pidfile /var/run/redis_<port>
# Data/Working Directory where the DB will be written inside this directory, with the filename # specified using the 'dbfilename' configuration directive
dir /<REDIS DATA PARTITION>/<REDIS PORT>/data

# Initially if we don’t have list of server ips for trusted access for cluster node.
bind 0.0.0.0
# bind server1_ip server2_ip ... servern_ip

# By default Redis does not run as a daemon. Use 'yes' as we need it.
daemonize yes

# Set log level to moderate
loglevel notice

cluster-enabled yes

cluster-config-file nodes.conf

cluster-node-timeout 5000

appendonly yes
```

---

## Create redis cluster ##

1. Cd redis installation path
2. Create redis cluster 

> ./redis-cli --cluster create itsrhv22205.it.statestr.com:6380 itsrhv17064.it.statestr.com:6379 itsrhv17065.it.statestr.com:6379

3. View the topology of redis cluster

> ./redis-cli -c -h {{redis_ipaddr}} -p {{redis_port}} -a {{Auth_password}} cluster nodes

---

## Add redis slave instance into redis cluster ##

1. Cd redis installation path
2. Add redis slave instance into redis cluster

> ./redis-cli --cluster add-node {{slave_ipaddr}}:{{slave_redis_port}} {{attached_master_ip}}:{{attached_master_port}} --cluster-slave --cluster-master-id {{master_node_uuid}} -a {{auth_password}}

3. View the topology of redis cluster

> ./redis-cli -c -h {{redis_ipaddr}} -p {{redis_port}} -a {{Auth_password}} cluster nodes

---

## Add redis master instance into redis cluster ##

1. Cd redis installation path
2. Add redis master instance into redis cluster

> ./redis-cli --cluster add-node {{master_ipaddress}}:{{master_port}} {{redis_instance_ip}}:{{redis_instance_port}} -a {{auth_password}}

3. Reshard slots from existed master nodes to newly added master node

> ./redis-cli --cluster reshard {{redis_instance_ip}}:{{redis_instance_port}} --cluster-from all --cluster-to {{current_instance_id}} --cluster-slots {{reshard_slot_num}} --cluster-yes -a {{auth_password}}

3. View the topology of redis cluster

> ./redis-cli -c -h {{redis_ipaddr}} -p {{redis_port}} -a {{Auth_password}} cluster nodes