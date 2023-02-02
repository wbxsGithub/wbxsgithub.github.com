---
title: clickhouse写入分布式表流程
date: 2023-02-02 19:18:58
tags:
---

### 前言

​	分布式表是ClickHouse非常重要的一部分，多用于数据的查询，但分布式表的写入流程对大家来讲可能不是很明确，下面对分布式表的写入流程进行分析。前面select的parser阶段和执行流程已经有大佬进行了分享，insert的解析大致与其相似，下面将略过解析流程直接从写入的核心部分接入。

### DistributedBlockOutputStream

```c++
DistributedBlockOutputStream(
        ContextPtr context_,
        StorageDistributed & storage_,
        const StorageMetadataPtr & metadata_snapshot_,
        const ASTPtr & query_ast_,
        const ClusterPtr & cluster_,
        bool insert_sync_,
        UInt64 insert_timeout_,
        StorageID main_table_);
```

从debug中可找到write的写入入口，这里就不贴图片了。

1.异步写入和同步写入

- 同步写入：数据直接写入实际的表中
- 异步写入：数据先被写入本地文件系统，然后异步发送到远端节点

```c++
void DistributedBlockOutputStream::write(const Block & block)
{
		....
    //主要判断逻辑
    if (insert_sync)
        writeSync(ordinary_block);
    else
        writeAsync(ordinary_block);
}
```

是否同步写入由insert_sync决定，而insert_sync的值由配置中的insert_distributed_sync决定，默认为false，即异步写入。

2.写入单shard还是所有shard

目前大多的集群都是采用默认的异步写入方式，下面就异步写入。分析写入本地节点和远端节点的情况。

```c++
void DistributedBlockOutputStream::writeAsyncImpl(const Block & block, size_t shard_id)
{
		...
		if (shard_info.hasInternalReplication())  //由 internal_replication 的设置决定
		{
				if (shard_info.isLocal() && settings.prefer_localhost_replica)
						writeToLocal(block, shard_info.getLocalNodeCount());
		}
		else
		{
				if (shard_info.isLocal() && settings.prefer_localhost_replica)
            writeToLocal(block, shard_info.getLocalNodeCount());
        std::vector<std::string> dir_names;
        for (const auto & address : cluster->getShardsAddresses()[shard_id])
            if (!address.is_local || !settings.prefer_localhost_replica)
            			dir_names.push_back(address.toFullString
            			(settings.use_compact_format_in_distributed_parts_names));
        if (!dir_names.empty())
            writeToShard(block, dir_names);
		}
}
```

实际上，同步和异步的写入的主要方法都是`writeAsyncImpl` ，其中writeToLocal方法是相同的，指在插入的目标中包含本地节点时优先选择本地节点写入。其中shard_info.hasInternalReplication()的判断，由internal_replication决定，是写入一个节点还是写入所有的节点都写一次。

3.写入分布式如何进行数据分发？

在writeToShard方法中，通过注册Monitor目录监听（requireDirectoryMonitor）实现数据的分发。

sharding_key的作用：通过sharding_key和设定的weight值来决定数据的写入策略。

具体的：在writeAsync和writeAsyncImpl之间存在方法：writeSplitAsync。当指定了sharding_key并且shard个数大于1时，则对block进行拆分。

将splitBlock分割的返回的splitted_blocks通过writeAsyncImpl方法写入。具体的写入方法writeToShard。

```c++
void DistributedBlockOutputStream::writeToShard(const Block & block, const std::vector<std::string> & dir_names)
{
    ...
    {
        const std::string path(disk_path + data_path + *it);
        const std::string tmp_path(path + "/tmp/");
        fs::create_directory(path);
        fs::create_directory(tmp_path);
        const std::string file_name(toString(storage.file_names_increment.get()) + ".bin");
        first_file_tmp_path = tmp_path + file_name;
        /// Write batch to temporary location
        {
            auto tmp_dir_sync_guard = make_directory_sync_guard(*it + "/tmp/");

            WriteBufferFromFile out{first_file_tmp_path};
            CompressedWriteBuffer compress{out, compression_codec};
            NativeBlockOutputStream stream{compress, DBMS_TCP_PROTOCOL_VERSION, block.cloneEmpty()};
            writeStringBinary(...);
            ...
            stream.writePrefix();
            stream.write(block);
            stream.writeSuffix();
        }
    }
  	/// Make hardlinks
    for (; it != dir_names.end(); ++it)
    {
        const std::string path(fs::path(disk_path) / (data_path + *it));
        fs::create_directory(path);

        const std::string block_file_path(fs::path(path) / (toString(storage.file_names_increment.get()) + ".bin"));
        createHardLink(first_file_tmp_path, block_file_path);
        auto dir_sync_guard = make_directory_sync_guard(*it);
    }
    ...
    storage.requireDirectoryMonitor(disk, dir_name, /* startup= */ false);
}
```

数据文件在本地写入的过程中会先写入tmp路径中，写完后通过硬链接link到shard目录，保证只要在shard目录中出现的数据文件都是完整写入的数据文件。
