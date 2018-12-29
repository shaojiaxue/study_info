&ensp;&ensp;随着服务器内存的增长和更加严格的低延迟需求，很多应用都决定将全部数据存储在内存中。在RocksDB中启动一个全内存的数据库非常简单，只需要将RocksDB数据目录mount到tmpfs or ramfs中即可。即使进程挂掉了，RocksDB也可以从in-memory文件系统中恢复所有的数据。但是，如果机器重启了会发生什么呢？
&ensp;&ensp;接下来会详细讲述在服务器重启后怎么恢复in-memory RocksDB的全部数据。
&ensp;&ensp;RocksDB的每一次update都会写入两个位置，一个是内存数据结构即memtable，另一个是WAL。WAL可以用来完全恢复memtable中的数据。默认情况下，当把内存表中的数据flush到SST file后，对应的WAL就会被删除，因为不需要再用这个WAL来恢复memtable（已经持久化）了。但是，如果SST file存储在in-memory file system里，那么就需要这个WAL日志在机器重启后来恢复日志。
&ensp;&ensp;Options::wal_dir是RocksDB存储WAL文件的目录。如果将这个目录配置在flash or disk，机器重启后并不会丢失当前的日志文件。Options::WAL_ttl_seconds 是指这些归档后的日志文件经历多长时间后被删除。如果设置为非零值，无用的log文件会被move到Options::wal_dir目录下的archive/目录。只有当timeout之后，才会删除这些归档日志文件。
&ensp;&ensp;假设Options::wal_dir配置在持久化存储上，Options::WAL_ttl_seconds配置为一天。为了完全能够恢复DB，我们必须以多于每天一次的频率来备份数据库的快照信息（包括table files 和 metadata files）。RocksDB提供了简单的方法来支持backup 数据库的快照。
&ensp;&ensp;用户应该配置backup过程，避免backup 日志文件，因为这些文件已经安全地保存在持久化存储上了。配置： BackupableDBOptions::backup_log_files=false
&ensp;&ensp;默认的restore过程会清除DB和WAL目录的数据。由于在backup文件中没有log文件，所以要确保在恢复数据库过程中不要删除WAL目录中的log文件。当restore时，配置RestoreOptions::keep_log_file=true。这个配置会move所有归档的日志文件回到WAL目录，RocksDB就可以replay所有归档日志文件中的操作，重建in-memory 数据库的状态。
总之，步骤如下：
* 将DB目录设置为mount 到tmpfs或者ramfs的位置
* 设置Options::wal_log为持久化存储上的目录
* 设置Options::WAL_ttl_seconds为T second
* 每隔T/2 second，backup RocksDB ，产生snapshot文件，使用的配置需要设置:BackupableDBOptions::backup_log_files = false
* 当丢失数据时，使用配置 RestoreOptions::keep_log_file = true来从backup中恢复数据。
