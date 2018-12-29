  RocksDB用户可以通过Options类将配置信息传入引擎，除此之外，还可以以下其他方法设置，分别为：
1. 通过option file生成一个option class
2. 从option string中获取option 信息
3. 从string map中获取option信息
# option string
用户可以调用GetColumnFamilyOptionsFromString() or GetDBOptionsFromString()将option string 传入方法，就可以解析出string中的option信息，也可以使拥GetBlockBasedTableOptionsFromString() and GetPlainTableOptionsFromString() 方法获取表格中的配置信息。每个option信息在option string中以<option_name>:<option_value>传入，多个option之间以;分割。
例如：
```
table_factory=PlainTable;prefix_extractor=rocksdb.CappedPrefix.13;comparator=leveldb.BytewiseComparator;compression_per_level=kBZip2Compression:kBZip2Compression:kBZip2Compression:kNoCompression:kZlibCompression:kBZip2Compression:kSnappyCompression;max_bytes_for_level_base=986;bloom_locality=8016;target_file_size_base=4294976376;memtable_huge_page_size=2557;max_successive_merges=5497;max_sequential_skip_in_iterations=4294971408;arena_block_size=1893;target_file_size_multiplier=35;min_write_buffer_number_to_merge=9;max_write_buffer_number=84;write_buffer_size=1653;max_compaction_bytes=64;max_bytes_for_level_multiplier=60;memtable_factory=SkipListFactory;compression=kNoCompression;bottommost_compression=kDisableCompressionOption;min_partial_merge_operands=7576;level0_stop_writes_trigger=33;num_levels=99;level0_slowdown_writes_trigger=22;level0_file_num_compaction_trigger=14;compaction_filter=urxcqstuwnCompactionFilter;soft_rate_limit=530.615385;soft_pending_compaction_bytes_limit=0;max_write_buffer_number_to_maintain=84;verify_checksums_in_compaction=false;merge_operator=aabcxehazrMergeOperator;memtable_prefix_bloom_size_ratio=0.4642;memtable_insert_with_hint_prefix_extractor=rocksdb.CappedPrefix.13;paranoid_file_checks=true;force_consistency_checks=true;inplace_update_num_locks=7429;optimize_filters_for_hits=false;level_compaction_dynamic_level_bytes=false;inplace_update_support=false;compaction_style=kCompactionStyleFIFO;purge_redundant_kvs_while_flush=true;hard_pending_compaction_bytes_limit=0;disable_auto_compactions=false;report_bg_io_stats=true;compaction_filter_factory=mpudlojcujCompactionFilterFactory;
```
# options map
类似的，用户可以通过string map获取配置信息，需要调用以下函数：GetColumnFamilyOptionsFromMap(), GetDBOptionsFromMap(), GetBlockBasedTableOptionsFromMap() or GetPlainTableOptionsFromMap()。
例如：
```
 std::unordered_map<std::string, std::string> cf_options_map = {
      {"write_buffer_size", "1"},
      {"max_write_buffer_number", "2"},
      {"min_write_buffer_number_to_merge", "3"},
      {"max_write_buffer_number_to_maintain", "99"},
      {"compression", "kSnappyCompression"},
      {"compression_per_level",
       "kNoCompression:"
       "kSnappyCompression:"
       "kZlibCompression:"
       "kBZip2Compression:"
       "kLZ4Compression:"
       "kLZ4HCCompression:"
       "kXpressCompression:"
       "kZSTD:"
       "kZSTDNotFinalCompression"},
      {"bottommost_compression", "kLZ4Compression"},
      {"compression_opts", "4:5:6:7"},
      {"num_levels", "8"},
      {"level0_file_num_compaction_trigger", "8"},
      {"level0_slowdown_writes_trigger", "9"},
      {"level0_stop_writes_trigger", "10"},
      {"target_file_size_base", "12"},
      {"target_file_size_multiplier", "13"},
      {"max_bytes_for_level_base", "14"},
      {"level_compaction_dynamic_level_bytes", "true"},
      {"max_bytes_for_level_multiplier", "15.0"},
      {"max_bytes_for_level_multiplier_additional", "16:17:18"},
      {"max_compaction_bytes", "21"},
      {"soft_rate_limit", "1.1"},
      {"hard_rate_limit", "2.1"},
      {"hard_pending_compaction_bytes_limit", "211"},
      {"arena_block_size", "22"},
      {"disable_auto_compactions", "true"},
      {"compaction_style", "kCompactionStyleLevel"},
      {"verify_checksums_in_compaction", "false"},
      {"compaction_options_fifo", "23"},
      {"max_sequential_skip_in_iterations", "24"},
      {"inplace_update_support", "true"},
      {"report_bg_io_stats", "true"},
      {"compaction_measure_io_stats", "false"},
      {"inplace_update_num_locks", "25"},
      {"memtable_prefix_bloom_size_ratio", "0.26"},
      {"memtable_huge_page_size", "28"},
      {"bloom_locality", "29"},
      {"max_successive_merges", "30"},
      {"min_partial_merge_operands", "31"},
      {"prefix_extractor", "fixed:31"},
      {"optimize_filters_for_hits", "true"},
  };
```
# 可用配置
&ensp;&ensp;不论是在option string还是option map中，option name是目标类中的变量名，这些包括：DBOptions, ColumnFamilyOptions, BlockBasedTableOptions, or PlainTableOptions。DBOptions and ColumnFamilyOptions中的变量名和变量描述信息可以在options.h中找到，BlockBasedTableOptions, and PlainTableOptions中的变量信息可以在table.h中找到。
&ensp;&ensp;需要注意的是，尽管绝大部分的配置项都可以在option string和option map中支持，仍然有一些例外。RocksDB支持的所有配置项可以在db_options_type_info, cf_options_type_info and block_based_table_type_info中查阅，源文件是util/options_helper.h。
&ensp;&ensp;如果配置项是一个回调类，比如:comparators, compaction filter, and merge operators，需要将回调类的指针作为option value传入。当然也有一些例外情况可以支持通过option string和option map传入回调类。
* Prefix extractor
option value可以这样传入：rocksdb.FixedPrefix.<prefix_length> or rocksdb.CappedPrefix.<prefix_length>
* Filter policy
option name：filter_poilcy
option value：bloomfilter:<bits_per_key>:<use_block_based>
* table factory
option name： table_factory
option value： BlockBasedTable or PlainTable
*Memtable Factory
option name： memtable_factory
option value ： skip_list/prefix_hash/hash_linkedlist/vector/cukoo
#option file
&ensp;&ensp;假设，我们打开了一个RocksDB实例，创建了一个列族，然后关闭实例，代码如下
```
s = DB::Open(rocksdb_options, path_to_db, &db);
...
// Create column family, and rocksdb will persist the options.
ColumnFamilyHandle* cf;
s = db->CreateColumnFamily(ColumnFamilyOptions(), "new_cf", &cf);
...
// close DB
delete cf;
delete db;
```
&ensp;&ensp;自从4.3版本以后，每个RocksDB实例都会自动存储最新的配置信息到一个配置文件中，这个配置文件可以在下次再打开数据库时使用。这和4.2版本或者更老版本差别还是很大的，在老版本，用户需要记录每个列族的配置信息，这样下次才可以成功打开DB实例。
&ensp;&ensp;首先，调用调用LoadLatestOptions()来加载目标RocksDB的最新配置信息
```
DBOptions loaded_db_opt;
std::vector<ColumnFamilyDescriptor> loaded_cf_descs;
LoadLatestOptions(path_to_db, Env::Default(), &loaded_db_opt,
                  &loaded_cf_descs);
```
&ensp;&ensp;由于c++没有映射机制，以下用户自定义的函数和类型指针必须在初始化时默认指定，更详细的信息可以在rocksdb/utilities/options_util.h找到。
```
* env
* memtable_factory
* compaction_filter_factory
* prefix_extractor
* comparator
* merge_operator
* compaction_filter
* cache in BlockBasedTableOptions
* table_factory other than BlockBasedTableFactory
```
&ensp;&ensp;这些不支持的用户自定义函数，开发者需要手动指定。例如：下面这个例子我们初始化BlockBasedTableOptions and CompactionFilter中的cache配置。
```
for (size_t i = 0; i < loaded_cf_descs.size(); ++i) {
  auto* loaded_bbt_opt = reinterpret_cast<BlockBasedTableOptions*>(
      loaded_cf_descs[0].options.table_factory->GetOptions());
  loaded_bbt_opt->block_cache = cache;
}

loaded_cf_descs[0].options.compaction_filter = new MyCompactionFilter();
```
&ensp;&ensp;接下来，我们可以做sanity check来保证可以正确安全地打开目标数据库
```
Status s = CheckOptionsCompatibility(
    kDBPath, Env::Default(), db_options, loaded_cf_descs);
```
&ensp;&ensp;如果check 返回OK，我们就可以继续使用上面load的配置集合来打开数据库了。
```
s = DB::Open(loaded_db_opt, kDBPath, loaded_cf_descs, &handles, &db);
```
&ensp;&ensp;RocksDB配置文件是INI 文件格式，每个配置文件都有一个版本区块，DBOption 区块、CFOption区块、TableOption区块，每一个列族都有这几类配置区块。以下是一个完整的配置文件示例
```
[Version]
  rocksdb_version=4.3.0
  options_file_version=1.1
[DBOptions]
  stats_dump_period_sec=600
  max_manifest_file_size=18446744073709551615
  bytes_per_sync=8388608
  delayed_write_rate=2097152
  WAL_ttl_seconds=0
  ...
[CFOptions "default"]
  compaction_style=kCompactionStyleLevel
  compaction_filter=nullptr
  num_levels=6
  table_factory=BlockBasedTable
  comparator=leveldb.BytewiseComparator
  compression_per_level=kNoCompression:kNoCompression:kNoCompression:kSnappyCompression:kSnappyCompression:kSnappyCompression
  ...
[TableOptions/BlockBasedTable "default"]
  format_version=2
  whole_key_filtering=true
  skip_table_builder_flush=false
  no_block_cache=false
  checksum=kCRC32c
  filter_policy=rocksdb.BuiltinBloomFilter
  ....
```
