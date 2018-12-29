&ensp;&ensp;当RocksDB收到一条Get()请求时，会依次从memtable、immutable memtable和SST files中去查找。SST files是按照层次组织的。
&ensp;&ensp;在level 0，文件是按照flush的时间戳顺序存储。每个file的key range（FileMetaData.smallest and FileMetaData.largest）大部分情况下都是有重叠的，所以需要查询每一个L0 file。
&ensp;&ensp;compaction会定期执行，从Ln层选择SST files，然后merge到一起生成一个新的SST file，然后下推到Ln+1层。所以，key/value s从L0依次沿着LSM tree 下降到Ln层。执行compaction操作时会把key/value s进行排序，然后拆分到多个文件中。从level 1到level n,SST files按照key 排序，且每个文件的key range互相不重叠。为了check一个key可能存在于哪一个一个SST file中，RocksDB并没有依次遍历每一个SST file然后去检查key是否在这个file的key range 内，而是执行二分搜索算法（FileMetaData.largest ）去定位这个SST file。二者相比，复杂度从O(N)下降到了O(logn)。然而，针对某些极端情况，logn的复杂度仍然无法接受。比如：上层到下一层的文件的扇出率是10的话，则level 3就有1000个文件。要将一个key locate到一个特定的文件，需要执行10次比较操作。这对于每秒执行几百万次查询的in-memory database来说，是个很大的消耗。
&ensp;&ensp;针对这个问题，解决方法可以如下：LSM tree build成功后，每一个层次的SST file的位置是对齐的，甚至，相对于下一层次的文件的位置也是对齐的。基于次，我们可以缩小二分搜索的范围。
```
                                         file 1                                          file 2
                                      +----------+                                    +----------+
level 1:                              | 100, 200 |                                    | 300, 400 |
                                      +----------+                                    +----------+
           file 1     file 2      file 3      file 4       file 5       file 6       file 7       file 8
         +--------+ +--------+ +---------+ +----------+ +----------+ +----------+ +----------+ +----------+
level 2: | 40, 50 | | 60, 70 | | 95, 110 | | 150, 160 | | 210, 230 | | 290, 300 | | 310, 320 | | 410, 450 |
         +--------+ +--------+ +---------+ +----------+ +----------+ +----------+ +----------+ +----------+ 
```
&ensp;&ensp;如上图，Level 1有2个文件，level 2有8个文件。现在，我们要检索 key=80。基于FileMetaData.largest 的二分搜索可以得出file 1是候选文件。然后比较80与file 1的FileMetaData.smallest和FileMetaData.largest来判断80是否在file1的range中。比较结果是80 小于FileMetaData.smallest (100)，所以file 1不可能包含key 80。接下来去检索level 2,。通常情况下，我们会对level 2中的8个文件去执行二分查找。但是，由于我们已经知道了80 小于100且只有file 1到file3才有可能包含小于100的key，我们就可以很明确地排除其他文件而不去检索。此时，我们的检索范围就从8个文件缩减到了3个。
&ensp;&ensp;再看一个例子，现在要检索的是key=230。在level1上执行二分搜索可以将key先定位到 file2。然后将230与file 2的上下界进行比较，得出key 是小于file 2's FileMetaData.smallest 300。即使，此时我们没有在level 1中找到目标key，但是可以推断出目标key是在200和300之间。此时，level 2中key range不包含[200, 300]的SST file可以被排除。此时，我们只需要检索level 2中的file5 和file 6即可。
&ensp;&ensp;基于上面的方法，我们可以在compaction时提前在level 1构建一些指针，这些指针指向level 2的某个范围内的文件列表。比如，level 1中的file1 左指针指向level 2中的file 3，右指针指向level 2中的file4。File 2的左右指针分别指向level2 中的file 6和7。查找时，这些指针可以用来确定二分查找的文件范围。
&ensp;&ensp;benchmark中显示SST file Index优化提升了5%的look up qps。
