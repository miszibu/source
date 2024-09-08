---
title: Redis OOM 故障注入
date: 2024-07-20 23:59:25
tags: [Redis] 
---

1. Redis BenchMark
2. Redis 内存排查命令
3. Redis 内存检查脚本



<!--more-->

## Redis OOM 故障注入

### Redis BenchMark

```shell
# 1. 持续给Redis灌数据
redis-benchmark -p 9999 -t set -r 100000000 -l
# 2. 模拟输入缓冲区过大
redis-benchmark -p 9999 -q -c 10 -d 102400000 -n 10000000 -r 50000 -t set
# 3. 模拟输出缓冲区过大
redis-benchmark -p 9999 -t get -r 5000000 -n 10000000 -d 100 -c 1000 -P 500 -l
```



### Redis 内存排查命令

```bash
# 1. 快速查看Redis内存是否够用
redis-cli -p 9999 info memory |egrep '(used_memory_human|maxmemory_human|maxmemory_policy)'
# 2. 检查复制积压缓冲区使用情况
redis-cli -p 9999 memory stats|egrep -A 1 '(total.allocated|replication.backlog)'
# 3. 检查客户端输入缓冲区内存使用总量
redis-cli -p 9999 client list| awk 'BEGIN{sum=0} {sum+=substr($12,6);sum+=substr($13,11)}END{print sum}'
# 4. 检查客户端输入缓冲区各客户端连接的内存情况
redis-cli -p 9999 client list|awk '{print substr($12,6),$1,$12,$18}'|sort -nrk1,1 | cut -f1 -d" " --complement
# 5. 检查客户端输出缓冲区内存使用总量
redis-cli -p 9999 client list| awk 'BEGIN{sum=0} {sum+=substr($16,6)}END{print sum}'
# 6. 检查客户端输出缓冲区各客户端连接的内存使用排序
redis-cli -p 9999 client list|awk '{print substr($16,6),$1,$16,$18}'|sort -nrk1,1 | cut -f1 -d" " --complement |head -n10
# 7. 检查数据对象使用内存总量
redis-cli -p 9999 memory stats|grep -A 1 'dataset.bytes'
```



### Redis 内存检查脚本

```shell
#!/bin/bash

if [ $# -ne 1 ];then
    echo -e "\033[31mERROR\033[0m: A port must be provided."
    echo "eg: sh $0 6379"
    exit 1
fi

PORT=$1
CLI_BIN=/data/redis/bin/redis-cli
EXEC="$CLI_BIN -p $PORT "

# Define checking memory available.
checkMem(){
    USED_MEM=`$EXEC memory stats |grep -A 1 total.allocated|tail -n1`
    MAX_MEM=`$EXEC info memory|grep -w maxmemory|awk -F ':' '{print $2}'`
    MEM_POL=`$EXEC info memory|grep -w maxmemory_policy|awk -F ':' '{print $2}'`
    DATA_USE=`$EXEC memory stats|grep -A 1 'dataset.bytes'|tail -n1`
    EXCEPT_DATA=`$EXEC memory stats|grep -A 1 'overhead.total'|tail -n1`
    REPL_USE=`$EXEC memory stats|grep -A 1 'replication.backlog'|tail -n1`
    INOUT_USE=`$EXEC client list| awk 'BEGIN{sum=0} {sum+=substr($12,6);sum+=substr($13,11)}END{print sum}'`
    OUTPUT_USE=`$EXEC client list| awk 'BEGIN{sum=0} {sum+=substr($16,6)}END{print sum}'`

    STATUS=`$EXEC set actionsky 1`
    if [ "$STATUS" = 'OK' ];then
        echo -e "Redis当前内存是否可用: \033\033[32m${STATUS}\033[0m"
    else
        echo -e "Redis当前内存是否可用: \033\033[31m${STATUS}\033[0m"
    fi
    echo "Redis当前内存淘汰策略: $MEM_POL"
    echo "Redis当前已使用的内存(byte): $USED_MEM"
    echo "Redis当前最大内限限制(byte): $MAX_MEM"
    echo "Redis当前数据对象已使用内存(byte): $DATA_USE"
    echo "Redis当前除数据外总内存消耗(byte): $EXCEPT_DATA"
    echo "Redis当前复制积压缓存区消耗(byte): $REPL_USE"
    echo " "
    echo "Redis当前客户端输入缓存总消耗(byte): $INOUT_USE"
    echo "Redis当前客户端输入缓存各连接消耗(TOP10):"
    $EXEC client list|awk '{print substr($12,6),$1,$12,$18}'|sort -nrk1,1 | cut -f1 -d" " --complement
    echo " "
    echo "Redis当前客户端输出缓存总消耗(byte): $OUTPUT_USE"
    echo "Redis当前客户端输出缓存各连接消耗(TOP10):"
    $EXEC client list|awk '{print substr($16,6),$1,$16,$18}'|sort -nrk1,1 | cut -f1 -d" " --complement |head -n10
}
```



