&ensp;&ensp;Checkpoint是RocksDB的一个feature，主要支持对当前正在运行的数据库制作一个snapshot。Checkpoints是一个时间点上的snapshot。当使用Read-only模式打开的话，可以支持查询这个时间点上的数据，当使用Read-Write模式打开的话，可以作为一个可写的snapshot。Checkpoints可以作为全量或者新增备份的backup使用。
&ensp;&ensp;给定一个RocksDB数据库，Checkpoint功能可以创建一个满足数据一致性的快照。如果snapshot所在的文件系统和DB file所在的文件系统相同的话，SST files会被硬链接，否则，就要全部拷贝过去，manifest和CURRENT files也会被拷贝过去。另外，如果有多个列族的话，在checkpoint的start和end时间段内的log文件也会被拷贝过去，这么做的目的是提供所有列族数据的一致性snapshot。
&ensp;&ensp;在创建checkpoints之前，需要创建一个checkpoint对象。
```
Status Create(DB* db, Checkpoint** checkpoint_ptr);
```
&ensp;&ensp;给定一个checkpoint对象和目录，CreateCheckpoint 函数会在给定目录中创建数据库的满足数据一致性的snapshot。
```
Status CreateCheckpoint(const std::string& checkpoint_dir);
```
&ensp;&ensp;函数参数中的目录不应该存在，这个目录会在函数中创建，另外，目录必须是一个绝对路径。checkpoint可以用作DB的一个read-only copy，也可以用作一个standalone DB。当打开了read/write时，SST file仍然是硬链接，后续如果文件被废弃后，硬链接也会被删除掉。如果用户不再使用这个snapshot了，用户可以直接删除这个数据目录来删除一个snapshot。
&ensp;&ensp;checkpoints广泛应用于MyRocks的在线备份。MyRocks是MySQL使用RocksDB作为存储引擎的版本。
