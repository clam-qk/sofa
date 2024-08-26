# 数据库

{% hint style="info" %}
&#x20;sofa目前是用的Pebble, Pebble 是一个由 CockroachDB 团队开发，受 RocksDB 和 LevelDB 启发的键值存储系统。它专注于性能和 CockroachDB 的内部使用，继承了 RocksDB 的文件格式和一些特性，如范围删除逻辑删除、表级布隆过滤器和 MANIFEST 格式更新。Pebble 有意不追求在 RocksDB 中包含的所有功能，而是专门针对 CockroachDB 所需的用例和功能集是一个由 CockroachDB 团队开发，受 RocksDB 和 LevelDB 启发的键值存储系统。它专注于性能和 CockroachDB 的内部使用，继承了 RocksDB 的文件格式和一些特性，如范围删除逻辑删除、表级布隆过滤器和 MANIFEST 格式更新。
{% endhint %}

### Pebble DB[​](https://docs.evmos.org/protocol/evmos-cli/alternative-dbs#pebble-db) <a href="#pebble-db" id="pebble-db"></a>

#### Install `sofad` binary from source[​](https://docs.evmos.org/protocol/evmos-cli/alternative-dbs#install-evmosd-binary-from-source) <a href="#install-evmosd-binary-from-source" id="install-evmosd-binary-from-source"></a>

```
# compile and install the binary
COSMOS_BUILD_OPTIONS=pebbledb make install
```

Check the binary version has the `-pebbledb` suffix

```
❯ sofad version
v19.0.0-pebbledb
```

NOTE:you'll need to replace the cometbft-db dependency before installing the binary:

```
# cd into the directory where you have the Sofa protocol source code
cd sofa

# replace the cometbft-db dependency
go mod edit -replace github.com/cometbft/cometbft-db=github.com/notional-labs/cometbft-db@pebble
go mod tidy

# compile and install the binary
go install -ldflags "-w -s -X github.com/cosmos/cosmos-sdk/types.DBBackend=pebbledb \
 -X github.com/cosmos/cosmos-sdk/version.Version=$(git describe --tags)-pebbledb \
 -X github.com/cosmos/cosmos-sdk/version.Commit=$(git log -1 --format='%H')" -tags pebbledb ./...
```

#### Update configuration[​](https://docs.evmos.org/protocol/evmos-cli/alternative-dbs#update-configuration) <a href="#update-configuration" id="update-configuration"></a>

Make sure to update the `db_backend` configuration parameter in the `config.toml`:

```
db_backend = "pebbledb"
```

#### Build docker image[​](https://docs.evmos.org/protocol/evmos-cli/alternative-dbs#build-docker-image) <a href="#build-docker-image" id="build-docker-image"></a>

To build a docker image with the `sofad` binary compiled to use pebbledb, run the following command:

```
make build-docker-pebbledb
```

### Features <a href="#sofatechnologywhitepaperbak-guan-jian-te-xing" id="sofatechnologywhitepaperbak-guan-jian-te-xing"></a>

* 块级表（Block-based tables）
* 检查点（Checkpoints）
* 索引批处理（Indexed batches）
* 迭代器选项（Iterator options）
* 基于层级的压缩（Level-based compaction）
* 手动压缩（Manual compaction）
* 合并操作符（Merge operator）
* 前缀布隆过滤器（Prefix bloom filters）
* 范围删除逻辑删除（Range deletion tombstones）
* 反向迭代（Reverse iteration）
* SSTable 摄取（SSTable ingestion）
* 单条删除（Single delete）
* 快照（Snapshots）
* 表级布隆过滤器（Table-level bloom filters）

### Technical highlights <a href="#sofatechnologywhitepaperbak-ji-shu-liang-dian" id="sofatechnologywhitepaperbak-ji-shu-liang-dian"></a>

* 与 RocksDB 的双向兼容性，特别是与 RocksDB 6.2.1 版本。
* 专注于 CockroachDB 所需的用例和功能集，有意不追求在 RocksDB 中包含的所有功能。
* 多项改进，如更快的反向迭代、改进的提交流程、L0 子级别和 L0 的并发压缩等。
* 作为一个 LSM 键值存储，提供 Set、Merge、Delete 和 DeleteRange 操作，支持批处理和快照。
* 内部使用预写日志（WAL）和排序字符串表（sstables）存储数据，Memtable 由并发抢占式的跳表实现。
