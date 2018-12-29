&ensp;&ensp;Block Cache是RocksDB把数据缓存在内存中以提高读性能的一种方法。开发者可以创建一个cache对象并指明cache capacity，然后传入引擎中。cache对象可以在同一个进程中宫多个DB Instance使用，这样开发者就可以通过配置控制所有的cache使用。Block cache存储的是非压缩的数据块内容。用户也可以设置另外一个block cache来存储压缩数据块。读数据时首先从非压缩数据块cache中读数据、然后读压缩数据块cache。当Direct-IO打开的话，压缩数据库可以作为系统页缓存的替代。
&ensp;&ensp;RocksDB中有两种cache的实现方式，分别为LRUCache和CLockCache。这两种cache都会被分片，来降低锁压力。用户设置的容量平均分配给每个shard。默认情况下，每个cache都会被分片为64块，每块大小不小于512K字节。
# Usage
&ensp;&ensp;默认情况下，RocksDB使用LRU cache实现，大小为8M。需要定制化设置一个block cache时，可以调用NewLRUCache() or NewClockCache()来创建对象实例，并设置到block based table options。用户也可以使用自己的cache（需要实现cache 接口）。
```
std::shared_ptr<Cache> cache = NewLRUCache(capacity);
BlockBasedTableOptions table_options;
table_options.block_cache = cache;
Options options;
options.table_factory.reset(new BlockBasedTableFactory(table_options));
```
&ensp;&ensp;要设置压缩block cache:
```
table_options.block_cache_compressed = another_cache;
```
&ensp;&ensp; 如果block_cache设置为nullptr的话，RocksDB会创建默认的block cache。
&ensp;&ensp; 关闭block cache
```
table_options.no_block_cache = true;
```
# LRU Cache
&ensp;&ensp;默认情况，RocksDB使用LRU Cache，默认大小为8M。cache的每个分片都有自己的LRU list和hash表来查找使用。每个shard都有个mutex来控制数据并发访问。不管是数据查找还是数据写入，线程都要获取cache分片的锁。开发中也可以调用NewLRUCache()来创建一个LRU cache。这个函数提供了几个有用的配置项来设置cache：
* capacity
cache的总大小
* num_shard_bits
去cache key的多少字节来选择shard_id。cache将会被分片为2^num_shard_bits
* strict_capacity_limit
很少会出现block cache的size超过容量的情况，这种情况发生在持续不断的read or iteration 访问block cache，pinned blocks的总大小会超过容量。如果有更多的读请求将block数据写入block cache时，且strict_capacity_limit=false(default)，cache服务会不遵循容量限制并允许写入。如果host没有足够内存的话，就会导致DB instance OOM。如果将这个配置设置为true，就可以拒绝将更多的数据写入cache，fail掉那些read or iterator。这个参数配置是以shard为控制单元的，所以会出现某一个shard在capcity满时拒绝继续写入cache，而另一个shard仍然有extra unpinned space。
* high_pri_pool_ratio
为高优先级block预留的capacity 比例
# Clock Cache
&ensp;&ensp;ClockCache实现了CLOCK算法。CLOCK CACHE的每个shard都有一个cache entry的圆环list。算法会遍历圆环的所有entry寻找unspined entry来回收，但是如果上次scan操作这个entry被使用的话，也会有继续留在cache中的机会。寻找并回收entry使用tbb::concurrent_hash_map。
&ensp;&ensp;使用LRUCache的一个好处是有一把细粒度的锁。在LRUCache中，即使是查找操作也需要获取分片锁，因为有可能会更改LRU-list。在CLock cache中查找并不需要获取分片锁，只需要查找当前hash_map就可以了，只有在insert时需要获取分片锁。使用clock cache，相比于LRU cache，写吞吐有一定提升。
```
Threads Cache     Cache               ClockCache               LRUCache
        Size  Index/Filter Throughput(MB/s)   Hit Throughput(MB/s)    Hit
    32   2GB       yes               466.7  85.9%           433.7   86.5%
    32   2GB       no                529.9  72.7%           532.7   73.9%
    32  64GB       yes               649.9  99.9%           507.9   99.9%
    32  64GB       no                740.4  99.9%           662.8   99.9%
    16   2GB       yes               278.4  85.9%           283.4   86.5%
    16   2GB       no                318.6  72.7%           335.8   73.9%
    16  64GB       yes               391.9  99.9%           353.3   99.9%
    16  64GB       no                433.8  99.8%           419.4   99.8%
```
通过调用NewCLockCache()来串讲一个clock cache。为了使clock cache生效，RocksDB需要链接上Intel TBB库。当创建clock cache时，也有一些可以配置的信息。
* capacity
same as LRUCache
* num_shard_bits
same as LRUCache
* strict_capacity_limit
same as LRUCache
#Caching Index and Filter Blocks
&ensp;&ensp;默认情况下，index和filter blocks都缓存在block cache外部，而且用户不能控制需要多少内存来cache这个block，当然如果用户设置了max_open_files配置除外。开发者可以将index和filter block数据cache在block cache中，以便更好地控制内存占用。
&ensp;&ensp;要想把index和filter block数据cache 在block cache中
```
BlockBasedTableOptions table_options;
table_options.cache_index_and_filter_blocks = true;
```
&ensp;&ensp;如果将index和filter blocks数据cache 在block cache中，这些block就需要和数据block竞争留在cache中。尽管index和filter blocks在内存中比数据block访问地更频繁，但是存在完全竞争失败的场景。这不是符合预期的，因为index和filter blocks往往比数据block更大，而且具有更高优先级留在cache中。
# Simulated Cache
SimCache是当cache capacity或者shard num发生改变时预测cache hit的方法。SimCache封装了真正的Cache 对象，运行一个shadow LRU cache模仿具有同样capacity和shard num的cache服务，检测cache hit和miss。这个工具在下面这种情况很有用，比如：开发者打开了一个DB 实例，配置了4G的cache size，现在想知道如果将cache size调整到64G时的cache hit。
创建一个sim cache：
```
// This cache is the actual cache use by the DB.
std::shared_ptr<Cache> cache = NewLRUCache(capacity);
// This is the simulated cache.
std::shared_ptr<Cache> sim_cache = NewSimCache(cache, sim_capacity, sim_num_shard_bits);
BlockBasedTableOptions table_options;
table_options.block_cache = sim_cache;
```
# Statistics
如果blick cache 计数表不为空的话，可以通过Options.statistics来访问。
```
// total block cache misses
// REQUIRES: BLOCK_CACHE_MISS == BLOCK_CACHE_INDEX_MISS +
//                               BLOCK_CACHE_FILTER_MISS +
//                               BLOCK_CACHE_DATA_MISS;
BLOCK_CACHE_MISS = 0,
// total block cache hit
// REQUIRES: BLOCK_CACHE_HIT == BLOCK_CACHE_INDEX_HIT +
//                              BLOCK_CACHE_FILTER_HIT +
//                              BLOCK_CACHE_DATA_HIT;
BLOCK_CACHE_HIT,
// # of blocks added to block cache.
BLOCK_CACHE_ADD,
// # of failures when adding blocks to block cache.
BLOCK_CACHE_ADD_FAILURES,
// # of times cache miss when accessing index block from block cache.
BLOCK_CACHE_INDEX_MISS,
// # of times cache hit when accessing index block from block cache.
BLOCK_CACHE_INDEX_HIT,
// # of times cache miss when accessing filter block from block cache.
BLOCK_CACHE_FILTER_MISS,
// # of times cache hit when accessing filter block from block cache.
BLOCK_CACHE_FILTER_HIT,
// # of times cache miss when accessing data block from block cache.
BLOCK_CACHE_DATA_MISS,
// # of times cache hit when accessing data block from block cache.
BLOCK_CACHE_DATA_HIT,
// # of bytes read from cache.
BLOCK_CACHE_BYTES_READ,
// # of bytes written into cache.
BLOCK_CACHE_BYTES_WRITE
```
