&ensp;&ensp;MemTable是一种在内存中保存数据的数据结构，然后再在合适的时机，MemTable中的数据会flush到SST file中。MemTable既可以支持读服务也可以支持写服务，写操作会首先将数据写入Memtable，读操作在query SST files之前会首先从MemTable中query数据（因为MemTable中的数据一直是最新的）。一旦MemTable满了，就会转换为只读的不可改变的，然后会创建一个新的MemTable来提供新的写操作。后台线程负责将MemTable中的数据flush到SST file，然后这个MemTable就会被销毁。
&ensp;&ensp;重要的配置
* memtable_factory：memtable的工厂对象。通过这个工厂对象，用户可以改变memtable的底层实现并提供个性化的实现配置。
* write_buff_size ：单个内存表的大小限制
* db_write_buff_size： 所有列族的内存表总大小。这个配置可以管理内存表的总内存占用。
* write_buffer_manager : 这个配置不是管理所有memtable的总内存占用，而是，提供用户自定义的write buffer manager来管理整体的内存表内存使用。这个配置会覆盖db_write_buffer_size。
* max_write_buffer_number：内存表的最大个数

&ensp;&ensp;memtable的默认实现是skiplist。除了默认memtable实现外，用户也可以使用其他类型的实现方法比如 HashLinkList、HashSkipList or Vector 来提高查询性能。
# Skiplist MemTable
&ensp;&ensp;基于Skiplist的memtable在支持读、写、随机访问和顺序scan时提供了较好的性能。此外，还支持了一些其他实现不能支持的feature比如concurrent insert和 insert with hint。
# HashSkiplist MemTable
&ensp;&ensp;如其名，HashSkipList是在hash table中组织数据，hash table中的每个bucket都是一个skip list，HashLinkList也是在hash table中组织数据，但是每一个bucket是一个有序的单链表。这两种结构实现目的都是在执行query操作时可以减少比较次数。一种使用场景就是把这种memtable和PlainTable SST格式结合在一起，然后将数据保存在RAMFS中。
&ensp;&ensp;当执行检索或者插入一个key时，key的前缀可以通过Options.prefix_extractor来检索，之后就找到了相应的hash bucket。进入到 hash bucket内部后，使用全部的key数据来进行比较操作。使用hash实现的memtable的最大限制是：当在多个key前缀上执行scan操作需要执行copy和sort操作，非常慢且很耗内存。
# flush
在以下三种情况下，内存表的flush操作会被触发：
* 内存表大小超过了write_buffer_size
* 全部列族的所有内存表大小超过了db_write_buffer_size，或者wrtie_buffer_manager发出了flush的指令。这种情况下，最大的内存表会被选择进行flush操作。
* 全部的WAL文件大小超过max_total_wal_size。在这种场景下，内存中数据最老的内存表会被选择执行flush操作，然后这个内存表对应的WAL file会被回收。

所以，内存表也可以在未满时执行flush操作。这也是产生的SST file比对应的内存表小的一个原因，压缩是是另一个原因（内存表总的数据是没有压缩的，SST file是压缩过的）。
# Concurrent Insert
如果不支持concurrent insert to memtable的话，来自多个线程的concurrent 写会顺序地写入memtable。默认是打开concurrent insert to memtable，也可以通过设置allow_concurrent_memtable_write来关闭。
# Comparison
![F7AC6702-7706-4f33-84BA-4ECC4047F567.png](https://upload-images.jianshu.io/upload_images/8751318-1499865c7fd4603b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

