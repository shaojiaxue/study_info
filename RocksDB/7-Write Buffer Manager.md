&ensp;&ensp;Write buffer mnager帮助开发者管理列族或者DB instance的内存表的内存使用。
* 管理内存表的内存占用在阈值内
* 内存表的内存占用转移到block cache

&ensp;&ensp;Write buffer manager与rate_limiter和sst_file_manager类似。用户创建一个write buffer manager对象，传入 column family或者DBs的配置中。可以参考write_buffer_manager.h的注释部分来学习如何使用。
#Limit total memory of memtables
&ensp;&ensp;在创建write buffer manager对象时，内存限制的阈值就已经确定好了。RocksDB会按照这个阈值去管理整体的内存占用。
&ensp;&ensp;在5.6或者更高版本中，如果整体内存表使用超过了阈值的90%，就会触发正在写入的某一个column family的数据执行flush动作。如果DB instance实际内存占用超过了阈值，即使全部的内存表占用低于90%，那也会触发更加激进的flush动作。在5.6版本以前，只有在内存表内存占用的total超过阈值时才会触发flush。
&ensp;&ensp;在5.6版本及更新版本中，内存是按照arena分配的total内存计数的，即使这些内存不是被内存表使用。在5.6之前版本中，内存使用是按照内存表实际使用的内存
#Cost memory used in memtable to block cache
&ensp;&ensp;从5.6版本之后，用户可以将内存表的内存使用的占用转移到block cache。不管是否打开内存表的内存占用，都可以这样操作。
&ensp;&ensp;大部分情况下，block cache中实际使用的blocks远比block cache中的数据少很多，所以如果用户打开了这个feature后，block cache的容量会覆盖掉block cache和内存表的内存占用。如果用户打开了cache_index_and_filter_blocks的话，这三种内存占用都在block cache中。
&ensp;&ensp;具体实现如下，针对内存表分配的每一个1M内存，WriteBufferManager都会在block cache中put一个dummy 1M的entry，这样block cache就可以正确的计算内部占用，而且可以在需要时淘汰掉一些block以便腾出内存空间。如果内存表的内存占用降低了，WriteBufferManager也不会立马三除掉dummmy blocks，而是在后续慢慢地释放掉。这是因为内存表空间占用的up and down太正常不过了，RocksDB不需要对此太过敏感。
* 把使用的block cache传递给WriteBufferManager
* 把WriteBufferManager的参数传入RocksDB内存表占用的最大内存
* 把block cache的容量设置为 data blocks和memtables的内存占用总和
