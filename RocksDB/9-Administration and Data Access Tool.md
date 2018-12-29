# Ldb Tool
&ensp;&ensp;ldb命令行工具提供了不同的数据访问和数据库管理命令。下面列举了一些sample。
**数据访问示例**
```
$./ldb --db=/tmp/test_db --create_if_missing put a1 b1
    OK 


    $ ./ldb --db=/tmp/test_db get a1
    b1
 
    $ ./ldb --db=/tmp/test_db get a2
    Failed: NotFound:

    $ ./ldb --db=/tmp/test_db scan
    a1 : b1
 
    $ ./ldb --db=/tmp/test_db scan --hex
    0x6131 : 0x6231
 
    $ ./ldb --db=/tmp/test_db put --key_hex 0x6132 b2
    OK
 
    $ ./ldb --db=/tmp/test_db scan
    a1 : b1
    a2 : b2
 
    $ ./ldb --db=/tmp/test_db get --value_hex a2
    0x6232
 
    $ ./ldb --db=/tmp/test_db get --hex 0x6131
    0x6231
 
    $ ./ldb --db=/tmp/test_db batchput a3 b3 a4 b4
    OK
 
    $ ./ldb --db=/tmp/test_db scan
    a1 : b1
    a2 : b2
    a3 : b3
    a4 : b4
 
    $ ./ldb --db=/tmp/test_db batchput "multiple words key" "multiple words value"
    OK
 
    $ ./ldb --db=/tmp/test_db scan
    Created bg thread 0x7f4a1dbff700
    a1 : b1
    a2 : b2
    a3 : b3
    a4 : b4
    multiple words key : multiple words value
```
**以HEX格式dump leveldb数据库数据**
```
$ ./ldb --db=/tmp/test_db dump --hex > /tmp/dbdump
```
**从HEX格式dump数据load到新的leveldb数据库中**
```
$ cat /tmp/dbdump | ./ldb --db=/tmp/test_db_new load --hex --compression_type=bzip2 --block_size=65536 --create_if_missing --disable_wal
```
**compact 一个当前存在的leveldb数据库**
```
$ ./ldb --db=/tmp/test_db_new compact --compression_type=bzip2 --block_size=65536
```
可以通过命令行参数--column_family=<string>来指定query操作的列族，--try_load_options可以load DB的option file以打开DB。当操作DB时，建议总是打开这个参数设置。如果打开DB时使用了默认的option而不是DB的option file中的信息，那么有可能mess up LSM-tree，且后续无法自动恢复。
#SST dump tool
sst_dump tool可以dump数据然后分析SST file。在SST file上，sst_dump tool 提供了多个操作方法。
```
$ ./sst_dump
file or directory must be specified.

sst_dump --file=<data_dir_OR_sst_file> [--command=check|scan|raw]
    --file=<data_dir_OR_sst_file>
      Path to SST file or directory containing SST files

    --command=check|scan|raw|verify
        check: Iterate over entries in files but dont print anything except if an error is encounterd (default command)
        scan: Iterate over entries in files and print them to screen
        raw: Dump all the table contents to <file_name>_dump.txt
        verify: Iterate all the blocks in files verifying checksum to detect possible coruption but dont print anything except if a corruption is encountered
        recompress: reports the SST file size if recompressed with different
                    compression types

    --output_hex
      Can be combined with scan command to print the keys and values in Hex

    --from=<user_key>
      Key to start reading from when executing check|scan

    --to=<user_key>
      Key to stop reading at when executing check|scan

    --prefix=<user_key>
      Returns all keys with this prefix when executing check|scan
      Cannot be used in conjunction with --from

    --read_num=<num>
      Maximum number of entries to read when executing check|scan

    --verify_checksum
      Verify file checksum when executing check|scan

    --input_key_hex
      Can be combined with --from and --to to indicate that these values are encoded in Hex

    --show_properties
      Print table properties after iterating over the file when executing
      check|scan|raw

    --set_block_size=<block_size>
      Can be combined with --command=recompress to set the block size that will
      be used when trying different compression algorithms

    --compression_types=<comma-separated list of CompressionType members, e.g.,
      kSnappyCompression>
      Can be combined with --command=recompress to run recompression for this
      list of compression types

    --parse_internal_key=<0xKEY>
      Convenience option to parse an internal key on the command line. Dumps the
      internal key in hex format {'key' @ SN: type}
```
**dump sst file blocks**
```
./sst_dump --file=/path/to/sst/000829.sst --command=raw
```
这个命令会生成一个txt file，命名为/path/to/sst/000829.sst。这个文件会包含所有的Index blocks和data blocks且以hex 编码，也会包含table property相关的信息，footer details和meta index datails。
**Printing entries in SST file**
```
./sst_dump --file=/path/to/sst/000829.sst --command=scan --read_num=5
```
**下面命令会打印SST file的前5个key
```
'Key1' @ 5: 1 => Value1
'Key2' @ 2: 1 => Value2
'Key3' @ 4: 1 => Value3
'Key4' @ 3: 1 => Value4
'Key5' @ 1: 1 => Value5
```
输入格式可以抽象为
```
'<key>' @ <sequence number>: <type> => <value>
```
如果key中含有非ASCII码字符的话，屏幕无法显示，在这种情况下最好使用--output_hex
```
./sst_dump --file=/path/to/sst/000829.sst --command=scan --read_num=5 --output_hex
```
也可以设置读操作的起始和结束位置
```
./sst_dump --file=/path/to/sst/000829.sst --command=scan --from="key2" --to="key4"
```
也可以用16进制来传递from to参数
```
./sst_dump --file=/path/to/sst/000829.sst --command=scan --from="0x6B657932" --to="0x6B657934" --input_key_hex
```
check SST file
```
./sst_dump --file=/path/to/sst/000829.sst --command=check --verify_checksum
```
这个命令会遍历SST file的所有entry，但是不会打印任何信息除非发现了SST file存在问题。这个命令操作也会校验checksum。
打印SST file的property
```
./sst_dump --file=/path/to/sst/000829.sst --show_properties
```
这个命令会读SST file文件的property，并打印，输出如下
```
from [] to []
Process /path/to/sst/000829.sst
Sst file format: block-based
Table Properties:
------------------------------
  # data blocks: 26541
  # entries: 2283572
  raw key size: 264639191
  raw average key size: 115.888262
  raw value size: 26378342
  raw average value size: 11.551351
  data block size: 67110160
  index block size: 3620969
  filter block size: 0
  (estimated) table size: 70731129
  filter policy name: N/A
  # deleted keys: 571272
```
SST_dump可以用来在不同的压缩算法下check 文件的size
```
./sst_dump --file=/path/to/sst/000829.sst --show_compression_sizes
```
使用--show_compression_sizes参数 sst_dump会在内存中重建SST file，可以使用不同的压缩算法，然后report file size，输出如下
```
from [] to []
Process /path/to/sst/000829.sst
Sst file format: block-based
Block Size: 16384
Compression: kNoCompression Size: 103974700
Compression: kSnappyCompression Size: 103906223
Compression: kZlibCompression Size: 80602892
Compression: kBZip2Compression Size: 76250777
Compression: kLZ4Compression Size: 103905572
Compression: kLZ4HCCompression Size: 97234828
Compression: kZSTDNotFinalCompression Size: 79821573
```
这些文件是在内存中生成的，文件中block size都是16KB，block_size可以通过--set_block_size配置修改


