## Backup API
c++ api，请参考:**include/rocksdb/utilities/backupable_db.h**。核心模块是backup engine，其暴露了创建backup的简单接口、查询backup的相关信息以及从backup中恢复数据。backup engine主要有以下形式:1) 创建backup的BackupEngine 2)从backup恢复数据的BackupEngineReadOnly。每个engine都可以用来查询backup的信息。
## Creating and verifying a backup
RocksDB实现了一种backup DB的简单方式，还可以对backup做正确性校验。下面是一个简单示例:
```
 #include "rocksdb/db.h"
    #include "rocksdb/utilities/backupable_db.h"

    #include <vector>

    using namespace rocksdb;

    int main() {
        Options options;                                                                                  
        options.create_if_missing = true;                                                                 
        DB* db;
        Status s = DB::Open(options, "/tmp/rocksdb", &db);
        assert(s.ok());
        db->Put(...); // do your thing

        BackupEngine* backup_engine;
        s = BackupEngine::Open(Env::Default(), BackupableDBOptions("/tmp/rocksdb_backup"), &backup_engine);
        assert(s.ok());
        s = backup_engine->CreateNewBackup(db);
        assert(s.ok());
        db->Put(...); // make some more changes
        s = backup_engine->CreateNewBackup(db);
        assert(s.ok());

        std::vector<BackupInfo> backup_info;
        backup_engine->GetBackupInfo(&backup_info);

        // you can get IDs from backup_info if there are more than two
        s = backup_engine->VerifyBackup(1 /* ID */);
        assert(s.ok());
        s = backup_engine->VerifyBackup(2 /* ID */);
        assert(s.ok());
        delete db;
        delete backup_engine;
    }
```
&ensp;&ensp;例子中会对/tmp/rocksdb_backup数据创建两个backup。用户可以使用相同的backup engine来创建和校验多个backup。
&ensp;&ensp;正常情况下，backup数据是递增的(参考BackupableDBOptions::share_table_files)。用户可以使用BackupEngine::CreateNewBackup() 创建一个新的backup，且只有新增的数据才会copy到backup 目录中。
&ensp;&ensp;如果你已经保存了一些backup，就可以调用BackupEngine::GetBackupInfo()来查询所有的backup信息。每一个backup都有一个递增的ID来标识。
&ensp;&ensp;当调用BackupEngine::VerifyBackups()时，会check backup目录中文件的大小，并与db目录中相应的文件对比。但是，并不会对文件的checksum进行校验，因为这样需要读取所有的文件数据。调用BackupEngine::VerifyBackups()接口进行校验唯一适用的场景就是在执行完create backup之后，因为校验逻辑中使用了backup期间的一些状态信息。
## Restoring a backup
```
 #include "rocksdb/db.h"
    #include "rocksdb/utilities/backupable_db.h"

    using namespace rocksdb;

    int main() {
        BackupEngineReadOnly* backup_engine;
        Status s = BackupEngineReadOnly::Open(Env::Default(), BackupableDBOptions("/tmp/rocksdb_backup"), &backup_engine);
        assert(s.ok());
        backup_engine->RestoreDBFromBackup(1, "/tmp/rocksdb", "/tmp/rocksdb");
        delete backup_engine;
    }
```
示例中会把第一个backup数据恢复到/tmp/rocksdb。BackupEngineReadOnly::RestoreDBFromBackup()的第一个参数是backup ID，第二个参数是目标DB目录，第三个参数是日志文件的目标位置目录（DB目录和LOG目录可以不一致）。BackupEngineReadOnly::RestoreDBFromLatestBackup()接口会从最新的backup（ID最大的backup）中恢复数据到DB。恢复期间，会计算所有存储文件的checksum，并与backup期间存储的checksum值对比。如果检测到checksum不匹配，就会中断restore过程，返回Status::Corruption。如果要查询恢复的数据，需要打开所有恢复成功的数据库。
## Backup directory structure
```
/tmp/rocksdb_backup/
├── LATEST_BACKUP
├── meta
│   └── 1
├── private
│   └── 1
│       ├── CURRENT
│       ├── MANIFEST-000008
|       └── OPTIONS-000009
└── shared_checksum
    └── 000007_1498774076_590.sst
```
&ensp;&ensp;LATEST_BACKUP是一个记录了最大backup id的文件。这个文件用来查询最大的backup id，由于META信息中也包含了这个最大id，所以在RocksDB 5.0版本之后就删除了这个文件。
&ensp;&ensp;meta目录包含了meta文件，对应地记录了每一个backup的描述信息，文件名就是backup id。如果有三个backup分别为1、2、3，那么这个目录中就包含了文件名为1、2、3的文件，记录了对应backup的信息。
&ensp;&ensp;private目录有non-SST 文件，主要是options,current,manifest和WAL。如果不设置Options::share_table_files，那么SST file也会在这个目录中。
&ensp;&ensp;shared目录包含了SSTfile（前提是设置了Options::share_table_files且Options::share_files_with_checksum未设置）。在这个目录中文件与db目录中的文件名相同。所以所以，在这种情况下，只能备份单个RocksDB实例，否则文件名会冲突。
&ensp;&ensp;shared_checksum目录中包含了SST files（前提是设置了Options::share_table_files和Options::share_files_with_checksum）。文件名以DB目录中的文件名、size和checksum组成。这个文件名是唯一的，可以来自不同的RocksDB实例。
##Backup performance
&ensp;&ensp;backup engine的open函数的耗时与当前存在的backup的数目是正相关的，因为我们要initialize所有backup 的信息。如果backup数据是在remote文件系统（比如HDFS）且有很多backup，那么初始化backup engine会消耗一额外的网络传输时间。官方建议backup engine一直保持打开状态，不需要在每一次backup或者restore时都重新创建。
&ensp;&ensp;另一种加速backup engine 初始化的方法就是删除非必须的backup。可以通过调用PurgeOldBackups(N)函数要删除backups，其中N表示要保留多少个backup。除了top N newest backups保留外，其他的都被删除。用户也可以调用DeleteBackup(id)来删除任一个backup。
&ensp;&ensp;要知道，backup性能是由从Local db中读数据的速度和拷贝到backup目录的速度共同决定的。尽管用户可以使用不同的环境来读数据和拷贝数据，但是仍然可能存在读写瓶颈。比如，如果local db是HDD的话，尽管配置了更多的线程来做backup，但是未必就会有效果。这是因为此时的性能瓶颈是磁盘的读性能，很有可能早就已经饱和了。一个低配的HDFS集群也不能提供好的并行性能。但是，如果local db是SSD且backup目标是在高性能的HDFS上的话，配置更多的线程往往会有收益。在RocksDB官方的benchmark里，配16个线程与单线程相比，前者的backup time是后者的1/3。
## Under the hood
当调用BackupEngine::CreateNewBackup()时会做以下工作：
* 1、禁止文件删除
* 2、找到live files（table、current、options、manifest）
* 3、copy live files到backup 目录
* 4、如果设置了flush_before_backup为false，需要拷贝log files到backup 目录。可以调用GetSortedWalFiles()然后拷贝到备份目录。
* 5、重新允许文件删除
## Advanced usage
&ensp;&ensp;RocksDB支持将用户自定义的metadata数据保存在backup中。调用BackupEngine::CreateNewBackupWithMetadata()函数可以创建backup并保存Metadata，后续可以调用BackupEngine::GetBackupInfo()来读取Metadata。这可以用来根据backup id查询meta信息来区分不同的backup。
&ensp;&ensp;RocksDB也支持备份和恢复options file。在恢复数据后，可以调用ocksdb::LoadLatestOptions() or rocksdb:: LoadOptionsFromFile()接口来从db目录中load配置信息。有个限制就是，并不是options 对象中的每个配置都可以转为text存储在文件中，在restore结束后加载options时，用户需要手动执行一些步骤来加载这些特殊的option信息。
&ensp;&ensp;备份时，需要实例化一些环境变量和初始化BackupableDBOptions::backup_env。设置backup root dir（BackupableDBOptions::backup_dir）。在backup目录中，文件会按照上述所示的结构组织在一起。
* BackupableDBOptions::share_table_files
这个配置控制着多个backup是否是递增完成的。如果设置为true的话，SST file都会存储在shred/目录下。如果不同的SST file具有相同名字的话，有可能会发生冲突（比如，多个数据库有相同的backup目录）。
* BackupableDBOptions::share_files_with_checksum
这个配置控制着shared files是怎么被识别的。如果设置为true的话，shared files是通过checksum、size和序列号来区分的。这样如果多个数据库配置了相同的backup目录的话，就会避免文件名冲突问题。
* BackupableDBOptions::max_background_operations
这个参数配置了在backup和restore期间，有多少个线程用于copy文件。对于HDFS等分布式分解系统，可以通过并行copy提高性能。
* BackupableDBOptions::info_log
是一个Logger object用来打印LOG 信息。

&ensp;&ensp;如果BackupableDBOptions::sync设置为true的话，RocksDB会在每一次文件写时调用fsync来同步文件数据和meta数据到磁盘，这样可以保证如果服务器宕机后重启的话，backups是满足数据一致性的。如果设置为false的话，可能会提高一点点性能，但是会引起backup的不一致问题。尽管如下，绝大部分情况下，还是没有问题的。
&ensp;&ensp;如果设置了BackupableDBOptions::destroy_old_data为true的话，创建一个新的BackupEngine时会删除所有老的backup数据。
&ensp;&ensp;BackupEngine::CreateNewBackup() 方法会有一个flush_before_backup参数，默认为false。如果这个参数为true的话，BackupEngine会首先执行一次memtable flush，然后只拷贝DB files 到backup 目录，不会拷贝log files到backup目录的原因是因为flush操作最终会触发并删除这些log file。如果这个参数设置为false的话，在开始backup时，BackupEngine不会执行flush操作。这种情况下，backup也会拷贝live memtable对应的log files。不管这个参数为true还是false，backup都会和数据库的当前状态保持一致性。
