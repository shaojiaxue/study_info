###Overview
&ensp;&ensp;Repairer会在RocksDB出现宕机等严重问题时尽最大努力去恢复尽可能多的数据，但是，并不能保证恢复数据库到一个一致性的状态。
###Usage
&ensp;&ensp;CLI命令行使用默认配置来修复DB，而且只会添加在SST file中发现的column family到repairer中。如果想自定义配置（自定义comparator、自定义column famify上的配置、自定义 column family set），那么就需要选择程序配置的方式。
#### 1 Programmatic
调用RepairDB的方法，具体声明在include/rocksdb/db.h
#### 2 CLI
要使用CLI，首先build ldb
```
$ make clean && make ldb
```
现在可以使用ldb的repair命令，指定DB。指令会打印info log到stderr，可以重定向到其他位子。这里启动修复DB（位置在./tmp，已经删除了MANIFEST file）。
```
$ ./ldb repair --db=./tmp 2>./repair-log.txt
$ tail -2 ./repair-log.txt 
[WARN] [db/repair.cc:208] **** Repaired rocksdb ./tmp; recovered 1 files; 926bytes. Some data may have been lost. ****
```
修复成功，MANIFEST文件已经恢复，DB也可以提供读服务
```
$ ls tmp/
000006.sst  CURRENT  IDENTITY  LOCK  LOG  LOG.old.1504116879407136  lost  MANIFEST-000001  MANIFEST-000003  OPTIONS-000005
$ ldb get a --db=./tmp
b
```
注意lost/目录，这个目录里是在recovery过程中丢掉的但是含有数据的一些文件。
### Repair Process
主要包括是个阶段：
* 找到files
repairer遍历数据目录中的所有文件，按照文件名进行分类。不能按照文件名进行分类的文件都会被忽略。
* 将logs转换为tables
每一个仍然live的log file都会replay。log file中的section如果checksum不匹配的话都会被跳过。repair过程中会有意地偏向保证数据一致性。
* 提取metadata
scan 每个table，计算：
1、smallest/largest for the table
2、largest sequence number in the table
如果无法scan table的file话，那就忽略这个table
* Write Descriptor
我们产生descriptor的内容如下：
1、log number is set to zero
2、next-file-number is set to 1 + largest file number we found
3、last-sequence-number is set to largest sequence# found across all tables
4、compaction pointers are cleared
5、every table file is added at level 0

### Possible optimization
1、计算出total size，选择max-level M
2、按照table中的最大sequence 对table进行排序
3、如果一个table与早一些的table有重叠的话，那就放在level-0，否则放在level-M
4、可以支持保证数据一致性的恢复和不安全的恢复（忽略checksum）
5、将每个表的metadata(smallest, largest, largest-seq#, ...)存储在表的meta section，以加速ScanTable操作。
