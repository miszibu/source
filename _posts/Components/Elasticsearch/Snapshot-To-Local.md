---
title: ElasticSearch Snapshot
date: 2020-04-16 16:43:42
tags: [Elasticsearch]
---

## Snapshot And Restore

Snapshot(快照)是正在运行的 ES Cluster 的一个备份。你可以选择部分 index，也可以选择整个集群来打 Snapshot，并将其存储在共享文件系统中，ES 支持例如S3,HDFS,Azure, Google Cloud Storage等等的远程仓库，只需要安装对应的 Plugin 就可以轻松集成。

Snapshot 是**增量备份的**，当我们为一个已经在仓库中存储过的 index做备份时，数据会在原有的基础上递增，因此我们可以为我们的集群频繁备份，不会产生冗余数据。

我们可以通过 **restore API** 来恢复Snapshot，当恢复 indices时，我们可以修改 index的名字和配置，十分灵活。

不建议通过直接拷贝 Data 文件夹的方式来拷贝数据，由于 ES Data 目录，是不断变更的，直接恢复可能会丢失数据或者恢复失败等等。**唯一可靠的备份数据的方式就是 Snaphot and Restore functionality**

<!--more-->

### Version compatibility

Snapshot 的版本兼容性的问题，实际上是 **Lucene Index 版本兼容性**的问题。**Snapshot 实际上存储了 index 在磁盘上的数据结构。**不同版本的 ES，在大版本之间都伴随着 Lucene 版本的升级，旧的 Lucene Index 是无法在新的 Lucene 上被读取的。

- A snapshot of an index created in 5.x can be restored to 6.x.
- A snapshot of an index created in 2.x can be restored to 5.x.
- A snapshot of an index created in 1.x can be restored to 2.x.

比如说，1.x版本的 Snapshot 自然是无法在5.x以后的 ES 中恢复的，两者内核 Lucene 的版本相差过大了。

Snapshot 可以存储不同版本的 Lucene Index，当我们全量恢复一个 Snapshot到目标集群时，如果其中部分 indices 创建于不兼容的版本时，我们就无法恢复 snapshot。因此在我们升级 ES Cluster之前，要确保 snapshot 不会存在不兼容的 indices 来影响集群的恢复。

当我们面临一个场景，需要从 snapshot中恢复不兼容的 indices ，怎么办呢？我们可以在一个支持恢复这个 Indices 的集群上先恢复 indices，然后再通过 reindex-from-remote 在远端 ES Cluster 上重建这个 index到当前版本。远程获取并恢复 indices 的操作相对耗时，当数据量较大时，可以先恢复部分数据，在充分考虑到时间影响后，再全部恢复。

### Repositories

在打 Snapshot 和 Restore 操作之前，我们需要实现注册一个 Snapshot Repository。我们**建议为每个 ES 大版本创建的对应的 Snapshot repository.**

Repository 存在不同的 Type. 比如当多个集群共享一个 Snapshot Repository 时，我们只会让一个集群具备 `Write` 权限，其他的集群只有 `readonly` 权限

Snapshot 的格式在不同的版本之间有所差别，因此当多个 ES Cluster 共享一个 Snapshot Repository时，可能会出现数据不可见，repository 崩溃等问题。

下面的例子就是来创建一个名为 my_backup 的 Repository。

```console
PUT /_snapshot/my_backup
{
  "type": "fs",
  "settings": {
    "location": "my_backup_location"
  }
}
```

使用 Get 请求来获取到已注册的 Snapshot Repository 信息

``` json
GET /_snapshot/my_backup
// 返回信息
{
  "my_backup": {
    "type": "fs",
    "settings": {
      "location": "my_backup_location"
    }
  }
}
// 使用通配符和,来批量获取 Repository 的信息
GET /_snapshot/repo*,*backup*

// 获取全部 snapshot repository 的信息，默认不声明时，就表示_all
GET /_snapshot
GET /_snapshot/_all
```



> Note: Snapshot Repository 的配置是非常重要的，我们需要配置Repo 的连接方式，如 URL, Directory Path, 外部组件的信息，还需要配置数据的存储方式，是否压缩，是否分块。还需要配置Type。文章不在这方面详细介绍，如果有实际应用，需要仔细了解下 Repository 配置。

### Snapshot

一个 Repository 可以存储多个 Snapshots。Snapshot之间以名字作为区分。

以下的命令将在 my_backup 这个Repository 中，创建名为 snapshot_1的 snapshot。

`wait_for_completion`参数表示，当 snapshot 初始化完成后，请求是否立即返回，默认情况是 false,请求会立即返回。设置为 True 后，请求将在 SnapShot 创建完成后才会返回。

```console
PUT /_snapshot/my_backup/snapshot_1?wait_for_completion=true
```

默认情况下，snapshot会对所有 Open and Started 的 indices 生效，我们也可以配置其对指定 indices 生效。

```console
PUT /_snapshot/my_backup/snapshot_2?wait_for_completion=true
{
  "indices": "index_1,index_2",
  "ignore_unavailable": true,
  "include_global_state": false
}
```

`ignore_unavailable`为 True 表示，当指定的 indices不存在时，跳过这 indices，默认情况下当indices 不存在时，请求将失败。



The list of indices that should be included into the snapshot can be specified using the `indices` parameter that supports [multi index syntax](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/multi-index.html). The snapshot request also supports the `ignore_unavailable` option. Setting it to `true` will cause indices that do not exist to be ignored during snapshot creation. By default, when `ignore_unavailable` option is not set and an index is missing the snapshot request will fail. By setting `include_global_state` to false it’s possible to prevent the cluster global state to be stored as part of the snapshot. By default, the entire snapshot will fail if one or more indices participating in the snapshot don’t have all primary shards available. This behaviour can be changed by setting `partial` to `true`.

Snapshot names can be automatically derived using [date math expressions](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/date-math-index-names.html), similarly as when creating new indices. Note that special characters need to be URI encoded.

使用以下命令创建一个以当前日期为结尾的 snapshot，如 `snapshot-2018.05.11`

```console
# PUT /_snapshot/my_backup/<snapshot-{now/d}>
PUT /_snapshot/my_backup/%3Csnapshot-%7Bnow%2Fd%7D%3E
```

The index snapshot process is incremental. In the process of making the index snapshot Elasticsearch analyses the list of the index files that are already stored in the repository and copies only files that were created or changed since the last snapshot. That allows multiple snapshots to be preserved in the repository in a compact form. Snapshotting process is executed in non-blocking fashion. All indexing and searching operation can continue to be executed against the index that is being snapshotted. However, a snapshot represents the point-in-time view of the index at the moment when snapshot was created, so no records that were added to the index after the snapshot process was started will be present in the snapshot. The snapshot process starts immediately for the primary shards that has been started and are not relocating at the moment. Before version 1.2.0, the snapshot operation fails if the cluster has any relocating or initializing primaries of indices participating in the snapshot. Starting with version 1.2.0, Elasticsearch waits for relocation or initialization of shards to complete before snapshotting them.

Besides creating a copy of each index the snapshot process can also store global cluster metadata, which includes persistent cluster settings and templates. The transient settings and registered snapshot repositories are not stored as part of the snapshot.

Only one snapshot process can be executed in the cluster at any time. While snapshot of a particular shard is being created this shard cannot be moved to another node, which can interfere with rebalancing process and allocation filtering. Elasticsearch will only be able to move a shard to another node (according to the current allocation filtering settings and rebalancing algorithm) once the snapshot is finished.

获取指定 snapshot 的信息：

```console
GET /_snapshot/my_backup/snapshot_1
```

This command returns basic information about the snapshot including start and end time, version of Elasticsearch that created the snapshot, the list of included indices, the current state of the snapshot and the list of failures that occurred during the snapshot. The snapshot `state` can be

| `IN_PROGRESS`  | The snapshot is currently running.                           |
| -------------- | ------------------------------------------------------------ |
| `SUCCESS`      | The snapshot finished and all shards were stored successfully. |
| `FAILED`       | The snapshot finished with an error and failed to store any data. |
| `PARTIAL`      | The global cluster state was stored, but data of at least one shard wasn’t stored successfully. The `failure` section in this case should contain more detailed information about shards that were not processed correctly. |
| `INCOMPATIBLE` | The snapshot was created with an old version of Elasticsearch and therefore is incompatible with the current version of the cluster. |

使用通配符来获取多个 Snapshot 的信息

```console
GET /_snapshot/my_backup/snapshot_*,some_other_snapshot
```

使用`_all`来获取一个 Repository 下的所有 snapshot 的信息。

All snapshots currently stored in the repository can be listed using the following command:

```console
GET /_snapshot/my_backup/_all
```

获取当前正在运行的 Snapshot 的信息

```console
GET /_snapshot/my_backup/_current
```

删除一个 Snapshot

```console
DELETE /_snapshot/my_backup/snapshot_2
```

Snapshot 被删除时，其所关联的文件也会被删除。当 Snapshot 正在创建时，Delete Snapshot 操作会打断它，当我们错误的创建了一个 Snapshot 时，直接删除即可。

注销 Repository，当一个 Repository 被注销时，ES 仅仅移除了这个 Repository 的引用，而 Repositroy 中包含的 Snapshot 仍然存在在文件系统中。

```console
DELETE /_snapshot/my_backup
```



### Restore

```console
POST /_snapshot/my_backup/snapshot_1/_restore
```

默认情况，snapshot 中的所有 indices 都会被恢复，而集群状态不会被恢复。

```console
POST /_snapshot/my_backup/snapshot_1/_restore
{
  "indices": "index_1,index_2",
  "ignore_unavailable": true,
  "include_global_state": true,
  "rename_pattern": "index_(.+)",
  "rename_replacement": "restored_index_$1"
}
```

`indices` 参数指定想要恢复的 indices，可以使用通配符

`include_global_state` 参数决定是否恢复 Cluster State, 默认false,不恢复

`rename_pattern` and `rename_replacement` 则决定是否要重命名 indices和如何重命名，以上述为例，符合 `rename_pattern`正则表达式的 indices 将会被重命名为 `rename_replacement`，$1指代的是原名。

`include_aliases`参数决定是否在恢复 indices 的同时，恢复其所关联的别名。

Restore 操作只能在正常运行的 ES Cluster 上执行。

1. **对应Index已经存在时，** 只能在 close 状态和当其与 snapshot 中的备份拥有相同 shards 数量的时候才能恢复。Restore 操作会将其打开。
2. 不存在的 Index，数据创建新的 index来恢复数据。
3. 当`include_global_state`被激活时，默认为 false，那么对应的 templates 如果不存在则创建，如果存在则替换。恢复的 Persistent Setting 会被直接添加。

#### Partial restore

在默认情况下，在 snapshot 恢复操作时，如果部分 indices 的 shard不可用，那么恢复操作将失败。

通过设置`partial`为 true，当存在不可用的 shards 时，将创建对应 empty shards，而不是失败。其他正常的 shards 仍会正常创建。

#### Changing index settings during restore

绝大多数的 Index Setting 可以在恢复的时候被修改，比如说，下面的例子中，index 会被恢复且不生成任何的 replicate shards且设置refersh_interval 值为默认。

```console
POST /_snapshot/my_backup/snapshot_1/_restore
{
  "indices": "index_1",
  "index_settings": {
    "index.number_of_replicas": 0
  },
  "ignore_index_settings": [
    "index.refresh_interval"
  ]
}
```

> Note: `index.number_of_shards`, primary shards number 是不可以改的

#### Restoring to a different cluster

Snapshot 并不跟集群紧密相关，因此一个集群创建的 snapshot 是可以给其他集群使用的，这里需要考虑的是版本兼容的问题。根据具体情况来说，自己把握。

在恢复的时候，还需要考虑到新集群的容量，架构等是否满足 indices 的要求，如果不满足我们可以在恢复Indices 时，修改 Replicate Number 操作，让 indices 恢复到一个相对较小的集群中去。

还要考虑的一点就是 Shard allocation filtering 了，我们会指定 shards 分配到有具体标签的节点中去，如果存在这个配置，我们就要修改他或者确保新集群能够满足这个配置，否则 indices 也会恢复失败的。

最后一点，在恢复 snapshot 时，默认也是会恢复 cluster status，这包括 templates, persistent setting 等等，在恢复时也会校验setting 能否应用新集群。比如当`discovery.zen.minimum_master_nodes`配置，就会影响新集群的 zen discovery。其实我个人建议直接不要恢复cluster status 就好了。

### Snapshot status

```console
// 获取当前正在运行的 snapshot 的详细信息。
GET /_snapshot/_status
// 获取指定 Repository 下的 snapshot 信息
GET /_snapshot/my_backup/_status
// 获取指定 Repository 指定 snapshot 的信息
GET /_snapshot/my_backup/snapshot_1/_status
```

返回json 示例：

```js
{
  "snapshots": [
    {
      "snapshot": "snapshot_1",
      "repository": "my_backup",
      "uuid": "XuBo4l4ISYiVg0nYUen9zg",
      "state": "SUCCESS",
      "include_global_state": true,
      "shards_stats": {
        "initializing": 0,
        "started": 0,
        "finalizing": 0,
        "done": 5,
        "failed": 0,
        "total": 5
      },
      "stats": {
        "incremental": {
          "file_count": 8,
          "size_in_bytes": 4704
        },
        "processed": {
          "file_count": 7,
          "size_in_bytes": 4254
        },
        "total": {
          "file_count": 8,
          "size_in_bytes": 4704
        },
        "start_time_in_millis": 1526280280355,
        "time_in_millis": 358,

        "number_of_files": 8,
        "processed_files": 8,
        "total_size_in_bytes": 4704,
        "processed_size_in_bytes": 4704
      }
    }
  ]
}
```

The output is composed of different sections. The `stats` sub-object provides details on the number and size of files that were snapshotted. As snapshots are incremental, copying only the Lucene segments that are not already in the repository, the `stats` object contains a `total` section for all the files that are referenced by the snapshot, as well as an `incremental` section for those files that actually needed to be copied over as part of the incremental snapshotting. In case of a snapshot that’s still in progress, there’s also a `processed` section that contains information about the files that are in the process of being copied.

*Note*: Properties `number_of_files`, `processed_files`, `total_size_in_bytes` and `processed_size_in_bytes` are used for backward compatibility reasons with older 5.x and 6.x versions. These fields will be removed in Elasticsearch v7.0.0.

同时返回多个 snapshot 信息

```console
GET /_snapshot/my_backup/snapshot_1,snapshot_2/_status
```

### Monitoring snapshot/restore progress



There are several ways to monitor the progress of the snapshot and restores processes while they are running. Both operations support `wait_for_completion` parameter that would block client until the operation is completed. This is the simplest method that can be used to get notified about operation completion.

The snapshot operation can be also monitored by periodic calls to the snapshot info:

```console
GET /_snapshot/my_backup/snapshot_1
```



Please note that snapshot info operation uses the same resources and thread pool as the snapshot operation. So, executing a snapshot info operation while large shards are being snapshotted can cause the snapshot info operation to wait for available resources before returning the result. On very large shards the wait time can be significant.

To get more immediate and complete information about snapshots the snapshot status command can be used instead:

```console
GET /_snapshot/my_backup/snapshot_1/_status
```



While snapshot info method returns only basic information about the snapshot in progress, the snapshot status returns complete breakdown of the current state for each shard participating in the snapshot.

The restore process piggybacks on the standard recovery mechanism of the Elasticsearch. As a result, standard recovery monitoring services can be used to monitor the state of restore. When restore operation is executed the cluster typically goes into `red` state. It happens because the restore operation starts with "recovering" primary shards of the restored indices. During this operation the primary shards become unavailable which manifests itself in the `red` cluster state. Once recovery of primary shards is completed Elasticsearch is switching to standard replication process that creates the required number of replicas at this moment cluster switches to the `yellow` state. Once all required replicas are created, the cluster switches to the `green` states.

The cluster health operation provides only a high level status of the restore process. It’s possible to get more detailed insight into the current state of the recovery process by using [indices recovery](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/indices-recovery.html) and [cat recovery](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/cat-recovery.html) APIs.

### Stopping currently running snapshot and restore operations

Snapshot and restore 框架 同时只支持一个 snapshot 和 restore 操作，若当前正在运行的 snapshot 有误或运行时间过长，我们可以通过 snapshot delete 操作来终止。

Sanpshot delete 操作会先检查 snapshot 是否运行结束，然后先停止正在运行的 snapshot，再从仓库中删除 snapshot 数据。

```console
DELETE /_snapshot/my_backup/snapshot_1
```

restore 操作使用标准的分片恢复机制，因此，任何对于当前正在恢复的索引删除操作，都会直接终止 restore 操作。

### Effect of cluster blocks on snapshot and restore operations



Many snapshot and restore operations are affected by cluster and index blocks. For example, registering and unregistering repositories require write global metadata access. The snapshot operation requires that all indices and their metadata as well as the global metadata were readable. The restore operation requires the global metadata to be writable, however the index level blocks are ignored during restore because indices are essentially recreated during restore. Please note that a repository content is not part of the cluster and therefore cluster blocks don’t affect internal repository operations such as listing or deleting snapshots from an already registered repository.

# Reference

[ESV6.7 snapshot](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/modules-snapshots.html)