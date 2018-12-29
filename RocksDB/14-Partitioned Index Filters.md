&ensp;&ensp;随着DB/mem使用越来越多，filter/index block的内存空间变得不可忽视。虽然cache_index_and_filter_blocks 配置只允许filter/index block数据的一部分cache在block cache中，但是还是会因为数据量的庞大影响RocksDB的性能。
* 占据了过多的block cache 空间，这些空间本来可以用于缓存data
* 当访问cache miss时需要load miss的数据到内存中，这无疑增大了磁盘存储的访问压力。

&ensp;&ensp;接下来会更详细地阐述这些问题的细节，并解释对Index/filter进行分片是怎么减轻这些开销的。
## How large are the index/filter blocks?
&ensp;&ensp;默认情况下，RocksDB每个SST file只有一个index/filter block。Index/filter的大小是由配置决定的，但是如果一个SST file为256MB的话，index/filter的block一般为0.5~5MB，这远比普通的data block（4~32KB）大很多。如果内存占用合适的话，每个SST file的index/filter只会读一次到内存即可，这样就不会总是与data block 竞争cache空间，不过也有可能会因为cache 淘汰导致多次从disk读取数据。
## What is the big deal with large index/filter blocks?
&ensp;&ensp;当index/filter block数据允许存储在Block cache时，就会与data block竞争cache这个非常稀缺的资源。5MB的index/filter就可以存储1000个data block（4KB），这将会导致更多data block的cache miss。多个SST file的index/filter彼此之间也会竞争cache并有可能会淘汰掉对方，也会加剧自身的cache miss。所以，导致了这样一种情况会出现：index/filter在cache 中的生命期间，真正提供cache服务的概率很低。
&ensp;&ensp;如果index/filter cache miss后，就需要从disk中reload，但是由于数据量很大，会导致很高的IO cost。一次简单的point lookup有可能需要两次data block（each for one layer of LSM）的读操作，所以有可能会读取很多MB的index/filter block数据。如果这种情况经常发生的话，disk cost就会更多消耗在index/filter 而不是data block上，这显然是我们不希望看到的。
## What is partitioned index/filters?
&ensp;&ensp;如果要分片的话，SST file的index/filter会被分片为多个小 block，并会配备一个索引。当需要读取index/filter时，只有top-level index会load到内存。然后，通过top-level index找出具体需要查询的那个分片，然后加载那个分片数据到block cache。top-level index占用的内存空间很小，可以存储在heap 也可以存储在block cache中，这取决于cache_index_and_filter_blocks配置。
### Pros
*  **更高的cache 命中率**:分片后，避免了超大的index/filter blocks 占用稀缺的cache空间，以更小的block形式加载需要的分片数据到block cache，这提高的内存空间的有效使用。
* **节省IO**: 当index/filter分片数据 cache miss后，只有一个分片需要从disk load到内存，与load SSTfile的全部index/filter相比，这会大大减轻disk的负载。
* **No compromise on index/filters**：如果没有采取分片策略的话，要想减缓index/filter内存空间占用的问题可以采取以下方法：设置更大的block或者减少bloom bits来使index/filter变得更小。前者会导致刚才所述的cache 浪费问题，后者会影响bloom filter功能的正确性。
### Cons
* **top-level index会占用额外空间**： index/filter大小的0.1~1%
* **更高的disk IO**：如果top-level index不在cache的话，会增加一次额外的IO。为了避免这种问题，可以将index 以更高的优先级存储在heap或者 cache中。（todo）
* **损失了空间局部性**: 如果是这样的场景，频繁且随机地读取相同SST文件的数据，这样就会在每次读取时都会load 不同的分片数据到内存，和一次性读取所有的index/filter相比，显然会更加低效。在RocksDB的benchmark中很少出现这种情况，但是这确实会发生在LSM tree的L0/L1数据访问中。因此，这两层的SST file 的index/filter可以不分片。(to do)
## 成功案例
### HDD, 100TB DB
DB大小为86G，HDD存储，在一个具有100TB数据的node上模拟小内存，使用Direct IO(关闭OS file cache)，block cache大小设置为60MB。分片后吞吐提升了11倍（ 5 op/s提升到55 op/s）。
```
/db_bench --benchmarks="readwhilewriting[X3],stats" 
--use_direct_reads=1
 -compaction_readahead_size 1048576 --use_existing_db --num=2000000000 --duration 600
 --cache_size=62914560 -cache_index_and_filter_blocks=false
 -statistics -histogram -bloom_bits=10 -target_file_size_base=268435456 
-block_size=32768 -threads 32 -partition_filters -partition_indexes 
-index_per_partition 100 -pin_l0_filter_and_index_blocks_in_cache
 -benchmark_write_rate_limit 204800
 -max_bytes_for_level_base 134217728 -cache_high_pri_pool_ratio 0.9
```
### SSD, Linkbench
DB大小300G，SSD存储，在相同node上模拟小内存（有可能会被其他的DB访问），打开Direct IO(关闭OS file cache)，block cache size设置为6G和2G。没有分片策略时，当把内存从6G降低到2G时，吞吐从38K tps降低到了23K。打开分片后，吞吐从38K降低到30K。
##How to use it?
* index_type = IndexType::kTwoLevelIndexSearch
这个配置是启用index分片
* NewBloomFilterPolicy(BITS, false)
使用full filters
* partition_filters = true
这个配置是启用filter分片
* metadata_block_size = 4096
index 分片的大小设置
* cache_index_and_filter_blocks = false [if you are on <= 5.14]
分片数据存储在cache中。控制top-level索引的存储位置，但是这种情况，在benchmark中实验数据不多。
* cache_index_and_filter_blocks = true and pin_top_level_index_and_filter = true [if you are on >= 5.15]
将所有的index/filter数据和top-level index都存储在block cache。
* cache_index_and_filter_blocks_with_high_priority = true
如字面意义
* pin_l0_filter_and_index_blocks_in_cache = true
1)   建议设置，因为这个配置会应用到index/filter 分片
2)   只在compation style 是level-based时使用
3)   需要注意：当把block 数据cache到block cache后，可能会导致超过内存设置的容量（如果strict_capacity_limit 没有设置）。

* block cache size: 如果之前都是将filter/index 存储在heap，现在设置filter/index 数据cache到block cache的话，不要忘了增加block cache size，大小与从heap中减少的量大概一致。
## Current limitations
* 如果没有对index分区的话，是不能对filters分区的。
* filter和index的partition 数量必须是一致的。换句话说，无论什么时候开始对index block进行切分，都要对filter block进行切分
* filter block的大小是由index block切分的时机决定的。RocksDB很快就会根据metadata_block_size 来控制filter和index block的最大size。换句话说：filter block切分发生在下面这两种情况，1)index block被切分了，所以会按照同样的分片数目切分filter block 2)filter block的size超过了metadata_block_size 
## Under the hood
#### 1 BlockBasedTable Format
分片之后index block存储由
```
[index block]
```
变换为:
```
[index block - partition 1]
[index block - partition 2]
...
[index block - partition N]
[index block - top-level index]
```

&ensp;&ensp;SST file的尾部是top-level index block，这个block本身就是partition blocks的索引。每一个index block 分片都是按照kBinarySearch格式存储。top-level index，也是按照这种格式存储。所以这些分片和索引数据可按照普通的data block reader来读取。
&ensp;&ensp;filter blocks也是按照相同的架构来分片。每个filter block都是按照KFullFilter格式存储。top-level index按照kBinarySearch格式存储，与index block一样。
&ensp;&ensp;如果分片的话，SST inspection工具 sst_dump不再汇报index/filter blocks的总大小，而是汇报index/filter的top-level index的大小。
#### 2 Builder
&ensp;&ensp;通过PartitionedIndexBuilder and PartitionedFilterBlockBuilder分别构建partitioned index和partitioned filter。
&ensp;&ensp;PartitionedIndexBuilder 有一个指针(sub_index_builder_)，指向ShortenedIndexBuilder，这个实例可以用来构建单钱的index 分片。当设置了flush_policy_时，PartitionedIndexBuilder 会将这个指针写入index block的最后一个key，然后创建一个新的ShortenedIndexBuilder。当调用了PartitionedIndexBuilder 的::Finish函数时，会在最早的sub index builder上调用::Finish函数，然后返回分片的block。下次调用PartitionedIndexBuilder::Finish时会携带上次返回的partition的offset信息，这个信息会被用作top-level index的值。最后一次调用PartitionedIndexBuilder::Finish会完成top-level index的构建。然后会将top-level index存储在SST file中，其offset会被用作index block的offset。
&ensp;&ensp;PartitionedFilterBlockBuilder 继承自FullFilterBlockBuilder ，都有一个FilterBitsBuilder 来构建bloom filters。PartitionedFilterBlockBuilder 有一个指针指向PartitionedIndexBuilder，可以调用其ShouldCutFilterBlock 函数来确定是否该对一个filter block进行切分。在分片时会首先调用FilterBitsBuilder ，将返回的block数据和一个由PartitionedIndexBuilder::GetPartitionKey()生成的一个partition key存储在一起，然后重置FilterBitsBuilder ，以供下次分片使用。最后，每调用一次PartitionedFilterBlockBuilder::Finish，都会返回一个partition以及当前partition用来构建top-level index的offset。最后一次调用::Finish会返回top-level索引的block。
&ensp;&ensp;之所以PartitionedFilterBlockBuilder 会依赖PartitionedIndexBuilder 是为了优化SST file的index/filter 分片。如果不care这个的话，后续改进中会将这个逻辑删除。
#### 3 Reader
&ensp;&ensp;PartitionIndexReader 可以通过读取top-level index block来获取分片索引信息。NewIterator 可以用作执行在top-level index的TwoLevelIterator 。这种简单的实现是可行的，因为每个index 分片都是kBinarySearch 格式，这和data block的格式相同，很容易就可以当做lower level iterator来使用。PartitionedFilterBlockReader 使用top-level index来找到filter partition的offset，然后在BlockBasedTable 对象上调用GetFilter()来加载FilterBlockReader 对象，然后释放掉FilterBlockReader 对象。
