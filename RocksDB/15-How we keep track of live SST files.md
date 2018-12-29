&ensp;&ensp;RocksDB内部有WAL log文件，此外，LSM tree还包含了一些SST files。每经过一次compaction，产出的新文件都会添加到SST files中，而输入的SST file会被删除。然而，compaction输入的SST file并不是立即就从SST file集合中删除，因为有可能在这些SST file上正进行着get or iterator操作。只有当冗余的SST file上没有任何操作的时候，才会执行真正的删除文件操作。接下来，我们会详细介绍怎么跟进SST file上的读写等操作信息。
&ensp;&ensp;LSM tree的文件信息保存在一个命名为version的数据结构中。当compaction or memtable flush结束后，SST file就会更新，此时，RocksDB就会为最新的LSM tree创建一个新的version来记录SST file信息。在某一个时刻，只会有一个“current” version数据结构来记录最新LSM tree的文件信息。get or iterator操作会使用这个version记录的文件来执行完get or iterator的过程。在get or iterator执行过程中涉及到的所有SST file都必须保留。如果一个version上没有任何get or iterator操作，那么这个version就会被drop掉，所有没有任何version记录的SST files也都会被删除掉。
&ensp;&ensp;刚开始时version记录了三个文件:
```
v1={f1, f2, f3} (current)
files on disk: f1, f2, f3
```
&ensp;&ensp;这时一个iterator操作使用了v1
```
v1={f1, f2, f3} (current, used by iterator1)
files on disk: f1, f2, f3
```
&ensp;&ensp;此时，发生了memtable flush,创建了一个新的version和 file 4
```
v2={f1, f2, f3, f4} (current)
v1={f1, f2, f3} (used by iterator1)
files on disk: f1, f2, f3, f4
```
&ensp;&ensp;执行了compaction操作，将f2 、f3、f4 compaction为 f5，产生最新的version3
```
v3={f1, f5} (current)
v2={f1, f2, f3, f4}
v1={f1, f2, f3} (used by iterator1)
files on disk: f1, f2, f3, f4, f5
```
&ensp;&ensp;我们可以看到，v2既不是最新的version，也没有任何读操作使用，所以v2以及f4将被删除。f1、f2、f3在v1中，v1仍然被iterator1使用，所以无法删除。
```
v3={f1, f5} (current)
v1={f1, f2, f3} (used by iterator1)
files on disk: f1, f2, f3, f5
```
&ensp;&ensp;加入此时，iterator1结束了，则最新状态为
```
v3={f1, f5} (current)
v1={f1, f2, f3}
files on disk: f1, f2, f3, f5
```
&ensp;&ensp;此时，v1既不是最新的version，也没有被任何操作引用，所以即将被删除，f2和f3也随之删除
```
v3={f1, f5} (current)
files on disk: f1, f5
```
&ensp;&ensp;这些逻辑是通过引用计数来实现的。每个SST file和version都有一个引用计数。当创建一个version时，会递增所有引用的SST file的引用计数。当一个version没有被使用时，它所包含的所有SST file的引用计数丢会递减1。如果一个文件的引用计数为0，那么就可以被删除了。
&ensp;&ensp;类似，每个version也都有一个引用计数。当创建一个version时，那这个version就是最新的version，引用计数递增1。如果一个version不再是最新的version时，其引用计数会递减1。在一个version上执行的任何操作都会将其引用计数递增1，这些操作计数后会将其引用计数递减1。当一个version的引用计数为0时，那么这个version就要被删除掉。如果一个version上有操作在执行或者是最新的version，那么他的引用计数就永远不会是0，也无法被删除。
&ensp;&ensp;有时候，reader操作户直接引用一个version，比如执行compaction操作时的输入version。更多情况下，reader是通过一个命名为super version的数据结构来引用version，这个super version会记录memtables和version的引用计数，super version其实就是a whole view of the DB。在使用super version来操作version的引用计数时，reader只需要对一个引用计数执行递增和递减操作就可以了。
&ensp;&ensp;RocksDB的所有version信息记录在VersionSet中，这个数据结构也同时记录了哪个version是当前version。由于每个column family都有一个单独的LSM树，所以每个column family都有一个version list（其中一个是当前version）。不过，每个DB中仅且只有一个VersionSet来记录所有column family的version s信息。
