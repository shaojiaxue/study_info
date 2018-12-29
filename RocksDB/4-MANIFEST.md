&ensp;&ensp;RocksDB对文件系统和存储介质是不可知的。文件系统的操作不是原子的，所以很有可能在系统故障时导致不一致。即使打开了日志记录功能，文件系统也不能在unclean restart时保证一致性。POSIX文件系统也不支持批量操作的原子性。所以，在RocksDB重启时，不能依靠存储在RocksDB data store file中的元信息来重建启动前的一致性状态。
&ensp;&ensp;RocksD有一个内建的机制来克服POSIX文件系统的各种限制，这种机制就是通过一个MANIFEST文件记录RocksDB状态改变的所有事物日志。所以，MANIFEST文件可以在DB重启时恢复到最近一次的一致性状态。
* MANIFEST
记录RocksDB状态改变的所有transction log的一种机制
* Manifest log
一个单独的文件，记录了RocksDB状态的snapshot和edit log
* CURRENT
最新最近的manifest log
# how does it work?
&ensp;&ensp;MANIFEST是RocksDB状态变更的transction log 记录。MANIFEST包含 manifest log文件和最新的manifest 文件指针。Manifest logs是滚动的日子文件，命名为MANIFEST-(序列号，递增)。CURRENT是一个特殊的文件用于指定latest manifest log 文件。
&ensp;&ensp;在系统启动或者重启时，latest manifest log文件包含了RocksDB的一致性状态。RocksDB后续的所有变更记录都写入这个manifest log 文件。当一个manifest log 文件超过了固定大小时，一个新的manifest文件会被创建，这个新的文件是基于当前DB的snapshot。CURRENT中的最新manifest 文件指针会更新，同时sync到文件系统中。一旦成功写入CURRENT file，冗余的manifest log文件会被清除。
```
MANIFEST = { CURRENT, MANIFEST-<seq-no>* } 
CURRENT = File pointer to the latest manifest log
MANIFEST-<seq no> = Contains snapshot of RocksDB state and subsequent modifications
```
# Version Edit
&ensp;&ensp;RocksDB在特定时间的特定状态会关联一个version。针对version的任何改动都被视为一个version edit。version（or RocksDB state snapshot）是通过join 一连串的version-edit构建起来。总之，一个manifest log文件是一连串的version edits。
```
version-edit      = Any RocksDB state change
version           = { version-edit* }
manifest-log-file = { version, version-edit* }
                  = { version-edit* }
```
#Version Edit Layout
&ensp;&ensp; Manifest log是一串version edit记录。version edit 记录的类型是通过 edit number指定。
## 1、Data Types
简单数据类型
```
VarX   - Variable character encoding of intX
FixedX - Fixed character encoding of intX
```
复杂数据类型
```
String - Length prefixed string data
+-----------+--------------------+
| size (n)  | content of string  |
+-----------+--------------------+
|<- Var32 ->|<-- n            -->|
```
## 2、Version Edit Record Format
version edit record 使用下面的格式，解码器通过Record ID来识别记录类型。
```
+-------------+------ ......... ----------+
| Record ID   | Variable size record data |
+-------------+------ .......... ---------+
<-- Var32 --->|<-- varies by type       -->
```
## 3、Version Edit Record Types and Layout
根据对RocksDB的状态变更的不同，有不同类型的edit record。
* Comparator edit record
```
Captures the comparator name

+-------------+----------------+
| kComparator | data           |
+-------------+----------------+
<-- Var32 --->|<-- String   -->|
```
* Log number edit record
```
Latest WAL log file number

+-------------+----------------+
| kLogNumber  | log number     |
+-------------+----------------+
<-- Var32 --->|<-- Var64    -->|
```
* Previous File Number edit record
```
Previous manifest file number

+------------------+----------------+
| kPrevFileNumber  | log number     |
+------------------+----------------+
<-- Var32      --->|<-- Var64    -->|
```
* Next File Number edit record
```
Next manifest file number

+------------------+----------------+
| kNextFileNumber  | log number     |
+------------------+----------------+
<-- Var32      --->|<-- Var64    -->|
```
* Last Sequence Number edit record
```
Last sequence number of RocksDB

+------------------+----------------+
| kLastSequence    | log number     |
+------------------+----------------+
<-- Var32      --->|<-- Var64    -->|
```
* Max Column Family edit record
```
Adjust the maximum number of family columns allowed.

+---------------------+----------------+
| kMaxColumnFamily    | log number     |
+---------------------+----------------+
<-- Var32         --->|<-- Var32    -->|
```
* Deleted File edit record
```
Mark a file as deleted from database.

+-----------------+-------------+--------------+
| kDeletedFile    | level       | file number  |
+-----------------+-------------+--------------+
<-- Var32     --->|<-- Var32 -->|<-- Var64  -->|
```
* 新文件的edit record
表明这个文件是新添加到数据库中的，同时提供RocksDB元信息。
&ensp;**a、File edit record with compaction information**
```
+--------------+-------------+--------------+------------+----------------+--------------+----------------+----------------+
| kNewFile4    | level       | file number  | file size  | smallest_key   | largest_key  | smallest_seqno | largest_seq_no |
+--------------+-------------+--------------+------------+----------------+--------------+----------------+----------------+
|<-- var32  -->|<-- var32 -->|<-- var64  -->|<-  var64 ->|<-- String   -->|<-- String -->|<-- var64    -->|<-- var64    -->|

+-----------+---------------+-------+------------------+-------+--------------+
|kPathID ---| Path size(n)  | path  | kNeedCompaction  | 1     | value (0/1)  |
+-----------+---------------+-------+------------------+-------+--------------+
<- var32  ->|<-- var32   -->|<- n ->|<-- var32      -->|<- 1 ->|<-- 1      -->|
```
&ensp;&ensp;**b、File edit record backward compatible**
```
+--------------+-------------+--------------+------------+----------------+--------------+----------------+----------------+
| kNewFile2    | level       | file number  | file size  | smallest_key   | largest_key  | smallest_seqno | largest_seq_no |
+--------------+-------------+--------------+------------+----------------+--------------+----------------+----------------+
<-- var32   -->|<-- var32 -->|<-- var64  -->|<-  var64 ->|<-- String   -->|<-- String -->|<-- var64    -->|<-- var64    -->|
```
&ensp;&ensp;**c、File edit record with path information**
```
+--------------+-------------+--------------+-------------+-------------+----------------+--------------+
| kNewFile3    | level       | file number  | Path ID     | file size   | smallest_key   | largest_key  |
+--------------+-------------+--------------+-------------+-------------+----------------+--------------+
|<-- var32  -->|<-- var32 -->|<-- var64  -->|<-- var32 -->|<-- var64 -->|<-- String   -->|<-- String -->|
+----------------+----------------+
| smallest_seqno | largest_seq_no |
+----------------+----------------+
<-- var64     -->|<-- var64    -->|
```
* Column family status edit record
```
Note the status of column family feature (enabled/disabled)

+------------------+----------------+
| kColumnFamily    | 0/1            |
+------------------+----------------+
<-- Var32      --->|<-- Var32    -->|
```
* Column family add edit record
```
Add a column family

+---------------------+----------------+
| kColumnFamilyAdd    | cf name        |
+---------------------+----------------+
<-- Var32         --->|<-- String   -->|
```
* Column family drop edit record
```
Drop all column family

+---------------------+
| kColumnFamilyDrop   |
+---------------------+
<-- Var32         --->|
```

