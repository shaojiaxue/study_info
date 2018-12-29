## What is a Bloom Filter?
&ensp;&ensp;在任意的keys集合中，应用一个算法并生成一个字节数组，这个字节数组就是Bloom filter。对于任意一个key，通过Bloom filter可以得出两个结论：1) 这个key有可能在集合中 2)这个key肯定不在集合中。
&ensp;&ensp;在RocksDB引擎中，如果设置了filter policy的话，每个新创建的SST file都会包含一个Bloom filter，这个Bloom filter可以确定我们要查找的key是否有可能在这个SST file中。Bloom filter其实就是一个bit array。在计算时，会使用多个hash 函数来计算指定的key，得到多个position，然后将bit array中相应的position 置为1。在使用Bloom filter时，对于一个key，仍然使用相同的hash函数来计算得出多个position，然后分别去check bit array中相应位置的值。只要有一个位置上的值不为1，则这个key 肯定不在集合中，否则，这个key就有可能在集合中（很大概率在集合中，除非集合超级超级大时才会有多个key映射到相同的n个位置）。
## LifeCycle
&ensp;&ensp;RocksDB中，每个SST file都有相应的一个Bloom filter，这个Bloom filter是在SST file写入存储时创建的，Bloom filter数据存储在相应的SST file中。其他各层的SST file都是用相同的方法生成Bloom filter。
&ensp;&ensp;Bloom filter只能从一个key集合中生成，所以我们不能combine多个Bloom filter。如果需要combine两个SST file，就会从combine之后的新的SST file中重新生成一个Bloom filter。
&ensp;&ensp;当打开一个SST file时，相应的Bloom filter也会被打开然后load到内存中。当SST file关闭时，对应的Bloom filt
er就会从内存中remove。要想cache Bloom filter到block cache中，使用:
```
BlockBasedTableOptions::cache_index_and_filter_blocks=true
```
##Block-based Bloom Filter (old format)
&ensp;&ensp;Bloom filter只能在所有的keys都在内存中时才能构建。换句话说，对一个bloom filter分片并不会影响误报率。所以为了在创建SSTfile时降低内存压力，老版本的实现是针对key-value数据的每一个2KB大小的block都单独生成一个bloom filter。最后，每个bloom filter的block数据的偏移量信息都存储在SST file中。
&ensp;&ensp;在读时，会首先从SST file的index中找到可能含有这个key-value对的block的offset。基于这个偏移量，相应的bloom filter会load到内存中。如果bloom filter表明这个key有可能存在，那么就去查找真实的数据块。
##Full Filters (new format)
&ensp;&ensp;老版本的filter block并不是内存对齐的，在lookup时有可能导致很多cache miss。尽管negative response（key不存在）避免了对数据块的检索，但是index数据还是需要load和检索的。新版本的实现--full filter 通过为每个SST file创建一个filter来解决这些问题。tradeoff就是为了cache SST file中的每一个key的hash值，需要消耗更多的内存。
&ensp;&ensp;Full filter limits the probe bits for a key to be all within the same CPU cache line。这可以限制CPU cache对每一个key的miss为1次来加速查询。Note that this is essentially sharding the bloom space and will not affect the false positive rate of the bloom as long as we have many key。
&ensp;&ensp;在读时，RocksDB使用创建SST file时相同的格式。用户可以设置filter_policy来指定新创建的SST file的格式。NewBloomFilterPolicy函数既可以创建老版本的filter也可以创建新版本的full filter。
```
extern const FilterPolicy* NewBloomFilterPolicy(int bits_per_key,
    bool use_block_based_builder = true);
}
```
####1 、Prefix vs. whole key
&ensp;&ensp;默认情况下，会对key的所有字节进行hash计算来设置bloom filter。这可以通过设置BlockBasedTableOptions::whole_key_filtering为false来避免对全部字节进行计算。当Options.prefix_extractor设置后，针对每个key的前缀计算的hash值也添加到了bloom filter中。由于key的前缀集合要小于key集合，因此计算key前缀生成的bloom filter会更小，当然也会提高误报率。
###2、Statistic
&ensp;&ensp;可以通过一些统计工具来看一看bloom filter在生成环境的执行情况。
如果支持prefix且设置了check_filter时，每次执行完:;Seek 和::SeekForPrev，都会更新以下统计值：
* rocksdb.bloom.filter.prefix.checked: seek_negatives + seek_positives
* rocksdb.bloom.filter.prefix.useful: seek_negatives
在每次执行完point lookup后，都会更新以下统计值。如果whole_key_filtering打开的话，结果是key全部字节数据的check 结果，否则就是key前缀数据的check结果。
* rocksdb.bloom.filter.useful: [true] negatives
* rocksdb.bloom.filter.full.positive: positives
* rocksdb.bloom.filter.full.true.positive: true positives
所以，误报率的计算为: 
(positives - true positives) / (positives - true positives + true negatives)
需要注意的是:
* 1、如果既设置了whole_key_filtering and prefix也设置了prefix，那么在point lookup时不会check prefix.
* 2、如果只设置了prefix，prefix bloom check的总次数是point lookup和seek之和。 Due to absence of true positive stats in seeks, we then cannot have the total false positive rate: only that of of point lookups。
## Customize your own FilterPolicy
Filter Policy支持用户自定义，主要的两个函数是：
```
FilterBitsBuilder* GetFilterBitsBuilder()
FilterBitsReader* GetFilterBitsReader(const Slice& contents)
```
new filter policy以一种提供FilterBitsBuilder和FilterBitsReader方法的工程形式工作。FilterBitsBuilder提供key 存储和filter生成的接口，FilterBitsReader提供在bloom filter中check key的接口。需要注意的是，这两个接口只适用于新版本的filter。
##Partitioned Bloom Filter
这是一种full filter的存储形式，可以把filter block 分片为多个更小的blocks，以降低block cache的压力。
##推导
计算false positive rate---FPR的推导过程如下：
* 假设bloom filter的byte array有m个bit，分为s个分片，则每个分片有m/s个bit.
* key集合元素个数为 n
* hash函数有k个，即每个key都可以计算出k个position
* 针对任意key，随机选择一个分片,且这个key的k个position 随机分布在分片内.
* 写入一个key后，如果分片的某一位仍是0，则可以分析如下：之前该key的hash后的位置都是在其他的分片上，概率为(s-1)/s，或者设置在这个分片的其他bit上，概率为1 - 1/(m/s)
**&ensp;&ensp;prob_0 = (s-1)/s + 1/s (1-s/m) ^ k**
* 二项式近似： (1+x)^k ~= 1 + xk  当  |xk| << 1 and |x| < 1 ，可以得出
**&ensp;&ensp;prob_0 = 1-1/s + 1/s (1-sk/m) = 1 - k/m = (1-1/m)^k**
* 近似估计， **prob_0(s=1)** is equal to **prob_0**.
* 插入n个key之后，某个bit仍然为0的概率：
**&ensp;&ensp;prob_0_n = (1-1/m)^kn**
* FPR 就是 所有k个bit为都是1（则该key有可能在集合中）:
**&ensp;&ensp;FPR = (1 - prob_0_n) ^ k = (1- (1-1/m)^kn) ^ k**

&ensp;&ensp;以上可知，FPR与s值无关，即FPR与bit array 分为多少个shard是无关的。换句话说，只要sk/m << 1,那么FPR就与分片无关（s为分片数、k为hash函数个数或者key对应的bit数，m为bit array大小）。
&ensp;&ensp;在full blooms，每个shard大小都是CPU高速缓存行的大小，这样就可以按照cpu高速缓存行对齐以降低 cpu cache miss。m将会被设置为： n * bits_per_key + epsilon，来保证是shard size的整数倍，比如:cpu cache line size。
