---
title: "Leveldb"
date: 2023-09-12T20:03:02+08:00
draft: false
toc: true
author: "李冬瓜"
tags: ["源码分享","服务端"]
categories: ["存储"]
keywords: ["源码","level实现原理"]
---
从KV应用场景入手，深入leveldb源码，学习KV存储引擎
<!--more-->
# LevelDB源码分享

>本文逻辑:
> 1. B+树缺点，LSM树优点、应用场景
> 2. LSM树结构、读写流程
> 3. LevelDB的存储组成：Journal(Log)、Memtable、磁盘文件结构
> 4. LevelDB读写流程、压缩流程
> 5. 扩展：leveldb思想的应用
>   
> 本文重点：
> 1. LevelDB的整体架构、核心存储结构
> 2. 读写流程、Compaction流程实现
> 3. LevelDB核心源码解析


## 背景
>MySQL关系型数据库使用B+树存储数据，笼统来讲适用读多写少场景；对于写入频繁场景，性能相对较差。

B+树有什么缺点
1. B+树非聚簇索引插入导致磁盘随机写
2. 逻辑上相近的叶子节点树物理上可能间隔较远，随机写消耗时间久；LSM保证顺序写
3. B+树可能有多个索引，插入一条数据需要同时更新对应索引。
4. B+树in-place-update，原地更新，比如插入数据可能导致页分裂。
5. B+树16K page存在空间利用率问题，可能有较多空闲块。

![img.png](/pic/leveldb/BTree.png)

如何设计更高效的写入数据结构一般都从以下三点来考虑：
1.  写磁盘优化为写内存
2.  随机写优化为顺序写
3. 单次写优化为批量写。

LSM树(Log-Structured-Merge-Tree)：一种常见的数据存储结构，广泛应用于键值存储引擎、分布式文件系统等领域；由Active Memtable、Immutable Memtable、WAL、SST等模块组成。  

LSM树优点：有效地规避了磁盘随机写入问题，但读取时可能需要访问较多的磁盘文件。 核心思想的核心就是放弃部分读能力，换取写入的最大化能力，放弃磁盘读性能来换取写的顺序性。  

LevelDB：基于LSM树的一种键值存储引擎。KV存储，适合写多读少、顺序读取、冷热数据分离场景  

RocksDB：基于LevelDB构建的高性能、持久化的键值存储引擎，对LevelDB进行了一系列优化和改进，进一步提高了性能和可靠性。  


>RocksDB和LevelDB之间有什么区别？
>1. 多线程压缩、锁优化、前缀bloom filter、memtable bloom filter、Column Families等
>2. 详见：https://github.com/facebook/rocksdb/wiki/Features-Not-in-LevelDB

>RocksDB的应用场景有哪些？
> 各种常用数据引擎

## LSM结构、读写流程  
> 下图为LSM读写核心流程，将贯穿全文始终，在后续对每个步骤进行拆解

![img_2.png](/pic/leveldb/lsm.png)

1. Journal(WAL)：Write Ahead Log，日志文件；用于防止因宕机等因素导致Memtable数据丢失。
2. Active MemTable：SkipList 实现的内存数据结构，新的数据修改会写入这里；写内存提升写入速度。
3. Immutable MemTable(Frozen Memtable)：当 Active MemTable 的大小达到设定的阈值时，会轮转变成 Immutable MemTable，只接受读操作；由后台线程接收compaction信号以Flush 到磁盘上，成为SST文件，异步落磁盘提升写入速度。
4. SST Files: Sorted String Table Files 排序字符串表文件，属于磁盘数据存储文件，一共分为 Level 0 到 Level N 多层，每一层包含多个 SST 文件，文件内数据有序，level 0 层内部文件之间可能会有交叉，其他层内部文件之间没有交叉；有序不重叠——提升读取速度，写入后数据不变更——不用加锁。
5. Block Cache：策略为LRU的缓存；缓存数据如：文件元信息、数据块等，加速读操作。
6. Manifest Files: 记录 SST 文件在不同 Level 的分布，单个 SST 文件的最大与最小key，已持久化的WAL日志文件序号及其他 LevelDB 元信息。
7. Current File：由于 Manifest 文件在系统启动、文件大小达到阈值时会新创建Manifest文件，由此导致文件变更, Current File记录当前生效的 Manifest 文件名。  

读流程：Active Memtable -> Immutable Memtable -> Level 0 SST Files -> Level 1~N SST Files  

写流程：Active Memtable ; Active Memtable -> Immutable Memtable (Rotate操作) ; Immutable Memtable -> SST Files  

> 写在前面：后续内容涉及较多源代码，优先看绿色重点关注部分，非绿色则不是重点； 

## LevelDB的存储组成  
> 从存储结构入手，对模块间关联具备整体认知。  

### Journal(WAL)  
![img_2.png](/pic/leveldb/lsm.png)

Journal介绍：图中红色部分，Write Ahead Log，一种日志文件；在写入Active Memtable前写入Journal进行持久化，用于防止因宕机等因素导致Memtable数据丢失。  

#### Journal结构与写入
Journal结构如下：

![img_2.png](/pic/leveldb/journal.png)

1. Block：为了增加读取效率，日志文件中按照block进行划分，每个block的大小为32KiB。每个block中包含了若干个完整的chunk。
   1. 由于一个block的大小为32KiB，因此当一条日志文件过大时，会将第一部分数据写在第一个block中，且类型为first，若剩余的数据仍然超过一个block的大小，则第二部分数据写在第二个block中，类型为middle，最后剩余的数据写在最后一个block中，类型为last。
2. Chunk：一条日志记录包含一个或多个chunk。
   1. Chunk分为Chunk Header(7字节) 和 Chunk Body
   2. Chunk Header：校验和(4字节)+Data长度(2字节)+ChunkType(1字节)
   3. Chunk Body：本日志的Batch Data
3. CRC校验和：不包含Length，包含ChunkType+本ChunkData计算的到的CRC校验和。
4. Length：Chunk Body-Data部分的长度
5. ChunkType：chunk共有四种类型：full，first，middle，last。一条日志记录若只包含一个chunk，则该chunk的类型为full。若一条日志记录包含多个chunk，则这些chunk的第一个类型为first, 最后一个类型为last，中间包含大于等于0个middle类型的chunk。
6. BatchData：实际用户操作的数据

写入JournalWriter 结构如下  
```go
package writer
    // Writer writes journals to an underlying io.Writer.
    type Writer struct {
    // w is the underlying writer.
    w io.Writer
    // seq is the sequence number of the current journal.
    seq int // 序列号
    // f is w as a flusher.
    f flusher
    // buf[i:j] is the bytes that will become the current chunk.
    // The low bound, i, includes the chunk header.
    i, j int // 待写入的左右边界
    // buf[:written] has already been written to w.
    // written is zero unless Flush has been called.
    written int // 已写入的数据
    // blockNumber is the zero based block number currently held in buf.
    blockNumber int64 // block序号，一个BlockSize为32KB
    // first is whether the current chunk is the first chunk of the journal.
    first bool // 当前chunk是否是第一个
    // pending is whether a chunk is buffered but not yet written.
    pending bool // 是否有构造完成的chunk缓存但未写入block
    // err is any accumulated error.
    err error
    // buf is the buffer.
    buf [blockSize]byte // 缓存
    }
```

#### BatchData
Chunk Body为写入Batch编码后的信息，结构如下：分别Batch Header 和 Batch Body  

Batch Header：Sequence Numer(8Bytes) + Batch Length(4Bytes)  

Batch Body：list<ValueType+KeyLength+Key+ValLength+Val>

![img_2.png](/pic/leveldb/batch_data.png)

### Memtable
#### SkipList
跳表：利用概率均衡技术，加快简化读写操作，且保证绝大大多操作均拥有O(log n)的良好效率。  

演变过程如下，单层顺序查找->多层近二分查找  
![img_2.png](/pic/leveldb/prove_skip_list.png)

查找流程：从头节点开始判断key与当前值和下一个值的大小，决定降低高度查找or本层遍历查找
![img_2.png](/pic/leveldb/skip_list_find.png)

写入流程：类似查找过程
1. 在查找的过程中，不断记录每一层的前任节点，如图中红色圆圈所表示的；
2. 为新插入的节点随机产生层高；
3. 在合适的位置插入新节点（例如图中节点12与节点19之间），并依据查找时记录的前任节点信息，在每一层中，以链表插入的方式，将该节点插入到每一层的链接中。
  
![img_2.png](/pic/leveldb/skip_list_insert.png)

#### Memtable实现
内存中基于跳表实现的数据结构，新的修改会写入active memtable，active memtable 大小达到阈值(默认4MB)之后会rotate成frozen memtable。  
>为什么用跳表而不用平衡树？ 
> 1. 平衡树为了维持树结构的平衡，获取稳定的查询效率，平衡树每次插入可能会涉及到复杂的节点旋转等操作。 
> 2. 设计跳表的目的是借助概率平衡，来构建一个快速且简单的数据结构，取代平衡树。  

![img_2.png](/pic/leveldb/lsm.png)  
Memtable中Key是InternalKey，由用户定义的Key+SequenceNumber+KeyType组成，如下图
![img_2.png](/pic/leveldb/internal_key.png)   
Memdb在代码中结构如下：
```go
// DB is an in-memory key/value database.
type DB struct {
    cmp comparer.BasicComparer
    rnd *rand.Rand

    mu     sync.RWMutex
    kvData []byte // 纯KV对；跳表中按照internalKey升序排列，userKey越大iKey越大，序列号越大的iKey越小
    // Node data:
    // [0]         : KV offset
    // [1]         : Key length
    // [2]         : Value length
    // [3]         : Height
    // [3..height] : Next nodes
    nodeData  []int  // 存储KV偏移量、长度、下一个节点的offset
    prevNode  [tMaxHeight]int // 存储每一层的前任节点
    maxHeight int // 最大高度
    n         int // KV数量
    kvSize    int // KV总大小
}
```  
Memdb对应结构图  

![img_5.png](/pic/leveldb/memdb_storage.png)   

代码证明：跳表中按照internalKey升序排列，userKey越大iKey越大，序列号越大的iKey越小  
```go
// NOTE 原生key a<b->-1，相同key情况下，InternalKey+Kt 越大的就越小； 5+2+1 排在 5+1+1前面
func (icmp *iComparer) Compare(a, b []byte) int {
    x := icmp.uCompare(internalKey(a).ukey(), internalKey(b).ukey())
    if x == 0 {
       if m, n := internalKey(a).num(), internalKey(b).num(); m > n {
          return -1
       } else if m < n {
          return 1
       }
    }
    return x
}
```
查找过程：从memdb中找到此Key对应的数据，调用findGreaterOrEqual函数；  

返回参数：刚好大于等于此key的值、是否找到；UserKey相同，比我大的最近一个是在我可见范围内的。  
```go
func (p *DB) findGE(key []byte, prev bool) (int, bool) { // NOTE findGreaterOrEqual，
    node := 0
    h := p.maxHeight - 1
    for {
       next := p.nodeData[node+nNext+h]
       cmp := 1
       if next != 0 {
          o := p.nodeData[next]
          cmp = p.cmp.Compare(p.kvData[o:o+p.nodeData[next+nKey]], key)
       }
       if cmp < 0 {  // 下一个Node比当前还小
          // Keep searching in this list
          node = next
       } else {
          if prev {
             p.prevNode[h] = node
          } else if cmp == 0 {
             return next, true
          }
          if h == 0 {
             return next, cmp == 0 // NOTE 刚好大于等于此key的值、是否找到
          }
          h--
       }
    }
}
```

插入过程：更新Memdb中对应Key的数据  
```go
// Put sets the value for the given key. It overwrites any previous value
// for that key; a DB is not a multi-map.
//
// It is safe to modify the contents of the arguments after Put returns.
func (p *DB) Put(key []byte, value []byte) error {
    p.mu.Lock()
    defer p.mu.Unlock()

    // 找到已有数据并更新
    if node, exact := p.findGE(key, true); exact {
       kvOffset := len(p.kvData)
       p.kvData = append(p.kvData, key...)
       p.kvData = append(p.kvData, value...)
       p.nodeData[node] = kvOffset // NOTE 这里直接改原本位置KV的offset，指向新的append的位置；但是不用改节点的位置
       m := p.nodeData[node+nVal]
       p.nodeData[node+nVal] = len(value)
       p.kvSize += len(value) - m
       return nil
    }

    h := p.randHeight() // NOTE 通过随机算法得出跳表这个节点要插入的层高
    if h > p.maxHeight {
       for i := p.maxHeight; i < h; i++ {
          p.prevNode[i] = 0
       }
       p.maxHeight = h
    }

    kvOffset := len(p.kvData)
    p.kvData = append(p.kvData, key...)
    p.kvData = append(p.kvData, value...)
    // Node
    node := len(p.nodeData)
    p.nodeData = append(p.nodeData, kvOffset, len(key), len(value), h)
    for i, n := range p.prevNode[:h] {
       m := n + nNext + i
       p.nodeData = append(p.nodeData, p.nodeData[m]) // NOTE 把当前h以下的preNode指向的下个节点 添加到 末尾新append进来元素的对应offset位置上
       p.nodeData[m] = node
    }

    p.kvSize += len(key) + len(value)
    p.n++
    return nil
}
```

### SST磁盘文件结构
SST Files: Sorted String Table Files 排序字符串表文件，读取时会在此类文件找对应Key；属于磁盘数据存储文件，一共分为 Level 0 到 Level N 多层，
每一层包含多个 SST 文件，文件内数据有序，level 0 层内部文件之间可能会有交叉，其他层内部文件之间没有交叉；有序不重叠——提升读取速度，写入后数据不变更——不用加锁。  

SST Files在系统中位置如下红框图：
![img_2.png](/pic/leveldb/lsm.png)  

#### 总体结构
根据功能不同，leveldb在逻辑上将sstable分为：
1. footer: 用来存储meta index block及index block的索引信息；
2. index block：index block中用来存储每个data block的索引信息；二分查找Key可能出现的区域，提升读性能；
3. meta Index block: 用来存储filter block的索引信息（索引信息指在该sstable文件中的偏移量以及数据长度）；
4. filter block: 用来存储一些过滤器相关的数据（布隆过滤器），但是若用户不指定leveldb使用过滤器，leveldb在该block中不会存储任何内容；在读DataBlock前使用bloomFilter判断Key是否存在，提升性能；
5. data block: 用来存储key value数据对；

![img_2.png](/pic/leveldb/sst_overview.png)  
<!-- 有时图片会解析失败，需要展示完整路径pic/leveldb，而有时pic/xx.png就能解析成功，有点奇怪-->

#### Footer
用来存储meta index block及index block的索引信息；
包含：MetaIndexBlock Offset、MetaIndexBlock Size、IndexBlock Offset、IndexBlock Size、MagicNumber
![img_2.png](/pic/leveldb/footer_detail.png)  
打开文件时会读取footer以获取indexBlock、MetaIndexBlock的索引，启动系统Reader代码如下  
```go
// 创建本文件的Reader
func NewReader(f io.ReaderAt, size int64, fd storage.FileDesc, cache *cache.NamespaceGetter, bpool *util.BufferPool, o *opt.Options) (*Reader, error) {
    ...
    footerPos := size - footerLen
    var footer [footerLen]byte
    if _, err := r.reader.ReadAt(footer[:], footerPos); err != nil && err != io.EOF {
       return nil, err
    }
    if string(footer[footerLen-len(magic):footerLen]) != magic { // 判断魔数做校验
       r.err = r.newErrCorrupted(footerPos, footerLen, "table-footer", "bad magic number")
       return r, nil
    }

    var n int
    // Decode the metaindex block handle. NOTE MetaIndexBlock索引
    r.metaBH, n = decodeBlockHandle(footer[:])
    if n == 0 {
       r.err = r.newErrCorrupted(footerPos, footerLen, "table-footer", "bad metaindex block handle")
       return r, nil
    }

    // Decode the index block handle. NOTE IndexBlock索引
    r.indexBH, n = decodeBlockHandle(footer[n:])  
    if n == 0 {
       r.err = r.newErrCorrupted(footerPos, footerLen, "table-footer", "bad index block handle")
       return r, nil
    }
    ...
}
type blockHandle struct {
    offset, length uint64
}
```
#### DataBlock
DataBlock的用途：用来存储key value数据对，支持查询。  

包含：CRC校验码、压缩类型、Restart数组、Entry列表  

Entry列表中每一个Entry组成：KeySharedLen+KeyUnsharedLen+ValLen+KeyDelta+Val
>为什么有Shared Unshared区分：由于sstable中所有的kv对都是严格按key升序存储的，为了节省存储空间，leveldb并不会为每一对keyvalue对都存储完整的key值，
> 而是存储与上一个key非共享的部分，避免key重复内容的存储。
  
>问题：由共享Key前缀引申出问题——读末尾元素时，需要找到最前面的Key获取公共前缀，耗时久且提高存储消耗(一个Block只以1个key为基准做共享)
解决方案-引入Restart数组： 
> 1. 构造：每间隔若干个keyvalue对，将为该条记录重新存储一个完整的key。重复该过程（默认间隔值为16），每个重新存储完整key的点称之为Restart point。 
> 2. 查询： 
>    1. 查询Key时先在restart数组中二分查找确定Key所在大体区域。 
>    2. 再依次对区间内所有数据项逐项比较key值，进行细粒度地查找；

Restart部分组成：RestartInterval(每N个Key设置一个Point)+Restart数组长度+Restart数组(存储restart point的offset)  
![img_2.png](/pic/leveldb/datablock_detail.png)  
写入Block代码如下  
```go
// 将KV对转化成Entry结构写入buf中
func (w *blockWriter) append(key, value []byte) (err error) {
    nShared := 0
    // 更新restart数组
    if w.nEntries%w.restartInterval == 0 {
       w.restarts = append(w.restarts, uint32(w.buf.Len()))
    } else {
       nShared = sharedPrefixLen(w.prevKey, key)
    }
    n := binary.PutUvarint(w.scratch[0:], uint64(nShared))
    n += binary.PutUvarint(w.scratch[n:], uint64(len(key)-nShared))
    n += binary.PutUvarint(w.scratch[n:], uint64(len(value)))
    // 写入shared_key_len + key_len + val_len
    if _, err = w.buf.Write(w.scratch[:n]); err != nil {
       return err
    }
    // 非共享部分key
    if _, err = w.buf.Write(key[nShared:]); err != nil {
       return err
    }
    // val
    if _, err = w.buf.Write(value); err != nil {
       return err
    }
    w.prevKey = append(w.prevKey[:0], key...)
    w.nEntries++
    return nil
}
// 完成本block，将内存中之前构建的restart数组刷进buf中
func (w *blockWriter) finish() error {
    // Write restarts entry.
    if w.nEntries == 0 {
       // Must have at least one restart entry.
       w.restarts = append(w.restarts, 0)
    }
    w.restarts = append(w.restarts, uint32(len(w.restarts)))
    for _, x := range w.restarts {
       buf4 := w.buf.Alloc(4)
       binary.LittleEndian.PutUint32(buf4, x)
    }
    return nil
}
```

> 为什么把DataBlock放前面？ 
> 
> 答：对于LevelDB来讲不管是 IndexBlock、MetaIndexBlock、FilterBlock、DataBlock，都是Block存储KV对，只不过逻辑上用途不同，但存储结构是相似的。
> 都有CRC校验码+CompressionType+Restart机制(filterBlock没有Restart)，只不过不同Block的Restart Interval不同而已。


#### IndexBlock
存储每个data block的索引信息，查询时根据restart数组二分定位到最临近的DataBlock索引，再遍历精准找到本Key应该在的DataBlock，提升性能
1. 一个文件有多个DataBlock但只有一个IndexBlock。
2. 同属于Block结构，但RestartInterval=1，即不使用公共前缀的优化机制。
3. Entry中存储
   1. Key：LastKey比指向的DataBlock max key要大，比下一个block min Key小；
   2. Val：指向DataBlock的Offset和Size

![img_2.png](/pic/leveldb/index_block_detail.png)  

#### MetaIndexBlock
存储filter block的索引信息（索引信息指在该sstable文件中的偏移量以及数据长度）；  

如下图省略Block公共部分后，Entry内部默认就一个KV  

![img_2.png](/pic/leveldb/meta_index_block_detail.png)  

#### FilterBlock
用来存储一些过滤器相关的数据（布隆过滤器），但是若用户不指定leveldb使用过滤器，leveldb在该block中不会存储任何内容；
在读DataBlock前，通过DataBlock的Offset找到对应filter，使用此filter判断Key是否存在，提升性能；  

一个Data Block对应一个Filter，一个Filter可对应多个DataBlock
![img_2.png](/pic/leveldb/filter_block_detail.png)  

构造filterBlock代码如下  

```go
func (w *filterWriter) flush(offset uint64) {
    if w.generator == nil {
       return
    }
    // 写入Block数据大于2KB时会构造新的filter
    for x := int(offset / uint64(1<<w.baseLg)); x > len(w.offsets); {
       w.generate()
    }
}
// 根据当前已有的keyHash生成新的Filter 
func (w *filterWriter) generate() {
    // Record offset.
    w.offsets = append(w.offsets, uint32(w.buf.Len())) // 对应filter的offset加进数组
    // Generate filters.
    if w.nKeys > 0 {
       w.generator.Generate(&w.buf)
       w.nKeys = 0
    }
}
// Generate 已经拿到当前dataBlock所有存储key，开始构造过滤器
func (g *bloomFilterGenerator) Generate(b Buffer) {
    // Compute bloom filter size (in both bits and bytes)
    nBits := uint32(len(g.keyHashes) * g.n) // 根据key数量计算布隆过滤器长度
    // For small n, we can see a very high false positive rate.  Fix it
    // by enforcing a minimum bloom filter length.
    if nBits < 64 {
       nBits = 64
    }
    nBytes := (nBits + 7) / 8
    nBits = nBytes * 8

    dest := b.Alloc(int(nBytes) + 1)
    dest[nBytes] = g.k
    for _, kh := range g.keyHashes {
       delta := (kh >> 17) | (kh << 15) // Rotate right 17 bits
       for j := uint8(0); j < g.k; j++ {
          bitpos := kh % nBits
          dest[bitpos/8] |= (1 << (bitpos % 8))
          kh += delta
       }
    }

    g.keyHashes = g.keyHashes[:0]
}
// 构造完成本文件的filterBlock，向文件buf中写入
func (w *filterWriter) finish() error {
    if w.generator == nil {
       return nil
    }
    // Generate last keys.

    if w.nKeys > 0 {
       w.generate()
    }
    w.offsets = append(w.offsets, uint32(w.buf.Len())) // offset数组的长度
    for _, x := range w.offsets {
       buf4 := w.buf.Alloc(4)
       binary.LittleEndian.PutUint32(buf4, x)
    }
    return w.buf.WriteByte(byte(w.baseLg)) // 大小为N的BlockSize就生成一个新的Filter
}
```

## 总结
SST File总存储架构如下
![img_2.png](/pic/leveldb/sst_file_overview.png)  

## LevelDB核心流程
### 非事务写入流程  

如图中蓝线部分为写入链路涉及到的环节  

![img_2.png](/pic/leveldb/lsm.png)  

与代码相关联的写入流程如下
1. 拿到写锁，合并channel中写操作
2. 写入WAL
3. 写入memdb
4. 根据memdb buf使用大小判断是否要轮转
5. 写入结束  

#### 合并操作&构造Batch数据
1. 当前写操作被其他写请求合并，等待响应并返回；
2. 当前写操作拿到写锁，合并其他写请求、构造batch数据，写入完成后返回。
![img_2.png](/pic/leveldb/merge_batch_data.png)  

等待当前操作被合并，或等待至拿到写锁  
```go
// 判断当前操作是否允许合并
func (db *DB) Write(batch *Batch, wo *opt.WriteOptions) error {
    ...
    merge := !wo.GetNoWriteMerge() && !db.s.o.GetNoWriteMerge()
    sync := wo.GetSync() && !db.s.o.GetNoSync()

    // Acquire write lock.
    if merge {
       select {
       case db.writeMergeC <- writeMerge{sync: sync, batch: batch}:
          if <-db.writeMergedC { // NOTE 这里注意差了一个字符，两个channel不一样的
             // Write is merged.
             return <-db.writeAckC
          }
          // Write is not merged, the write lock is handed to us. Continue.
       case db.writeLockC <- struct{}{}: // NOTE 等merge或者等写锁；
          // Write lock acquired.
       }
    } else {
       ...错误处理
    }
    // NOTE 带着锁的写入
    return db.writeLocked(batch, nil, merge, sync)
}
```

拿到写锁以后合并其他写请求：从channel中取出写操作合并，无操作待合并或Batch操作超过长度限制则停止。  
```go
// 合并channel中操作，并执行，写WAL、Memdb、轮转memdb
// ourBatch is batch that we can modify.
func (db *DB) writeLocked(batch, ourBatch *Batch, merge, sync bool) error {
       // NOTE 和待merge的chan中操作进行merge；如果merge溢出则停止，走下一步实际写入
    merge:
       for mergeLimit > 0 {
          select {
          case incoming := <-db.writeMergeC:
             if incoming.batch != nil { // 是batch操作
                // Merge batch.
                if incoming.batch.internalLen > mergeLimit { // NOTE 超出merge限制
                   overflow = true
                   break merge
                }
                batches = append(batches, incoming.batch)
                mergeLimit -= incoming.batch.internalLen
             } else { // 是单个KV put操作
                // Merge put.
                internalLen := len(incoming.key) + len(incoming.value) + 8
                if internalLen > mergeLimit {
                   overflow = true
                   break merge
                }
                if ourBatch == nil { 
                   ourBatch = db.batchPool.Get().(*Batch)
                   ourBatch.Reset()
                   batches = append(batches, ourBatch)
                }
                // We can use same batch since concurrent write doesn't
                // guarantee write order.
                ourBatch.appendRec(incoming.keyType, incoming.key, incoming.value)
                mergeLimit -= internalLen
             }
             sync = sync || incoming.sync
             merged++
             db.writeMergedC <- true // NOTE 这里merge以后就直接塞进去了，不会等到写入进memtable再去塞chan

          default:
             break merge
          }
       }
    }

    // Release ourBatch if any.
    if ourBatch != nil {
       defer db.batchPool.Put(ourBatch)
    }

    db.unlockWrite(overflow, merged, nil)
    return nil
}
```

#### 写入WAL
1. 获取写入Journal文件的Writer
2. 判断当前缓存buffer是否超过大小，超过则写入Block并重置当前成员变量
3. 根据Batch数据编码log头部并写入
4. 写入Batch信息
5. Flush数据完成写入
   
![img_2.png](/pic/leveldb/write_wal.png)   

写入WAL操作展开代码如下：  
```go
func (db *DB) writeJournal(batches []*Batch, seq uint64, sync bool) error {
    wr, err := db.journal.Next() // 获取写入Writer
    if err != nil {
       return err
    }
    if err := writeBatchesWithHeader(wr, batches, seq); err != nil { // 写入log头、BatchData
       return err
    }
    if err := db.journal.Flush(); err != nil { 
       return err
    }
    if sync {
       return db.journalWriter.Sync()
    }
    return nil
}
```

#### 写入Memdb
1. 构造要写入memdb的KV
2. 从memdb(跳表)中查找此key，找到则直接更新数据
3. 找不到则在对应位置插入此key

![img_2.png](/pic/leveldb/write_memdb.png)   

#### 轮转Memdb
1. 在写入完成后，判断当前memdb是否超过容量限制，未超过则跳过此流程。
2. 当前已有frozen memdb，而leveldb允许同一时刻仅存在一个frozen memdb，向channel发送信号，等待当前frozen memdb落成SST File
3. 当前memdb轮转成frozen memdb并返回

![img_2.png](/pic/leveldb/rotate_memdb.png)   
将memdb转化成Frozen memdb，如果已有frozen memdb则等待其落磁盘，系统中同时仅能有一个frozen memdb存在  

#### 非事务写入流程汇总

1. 拿到写锁，合并channel中写操作
2. 写入WAL
3. 写入memdb
4. 根据memdb buf使用大小判断是否要轮转
5. 写入结束

![img_2.png](/pic/leveldb/write_overview.png)   

写入主流程核心代码：写WAL、写memdb、轮转memdb  
```go 

// 合并channel中操作，并执行，写WAL、Memdb、轮转memdb
// ourBatch is batch that we can modify.
func (db *DB) writeLocked(batch, ourBatch *Batch, merge, sync bool) error {
    // Release ourBatch if any.
    if ourBatch != nil {
       defer db.batchPool.Put(ourBatch)
    }

    // Seq number.
    seq := db.seq + 1

    // NOTE 先写WAL
    // Write journal.
    if err := db.writeJournal(batches, seq, sync); err != nil {
       db.unlockWrite(overflow, merged, err)
       return err
    }

    // NOTE 把操作写入内存memtable中
    // Put batches.
    for _, batch := range batches {
       if err := batch.putMem(seq, mdb.DB); err != nil {
          panic(err)
       }
       seq += uint64(batch.Len()) // 序列号代表操作的个数
    }

    // Incr seq number.
    db.addSeq(uint64(batchesLen(batches))) // 修改DB序列号

    // NOTE 如果当前的memdb超过阈值，需要进行轮转
    // Rotate memdb if it's reach the threshold.
    if batch.internalLen >= mdbFree {
       if _, err := db.rotateMem(0, false); err != nil {
          db.unlockWrite(overflow, merged, err)
          return err
       }
    }

    db.unlockWrite(overflow, merged, nil)
    return nil
}
```

### 读流程
读指定Key数据，大体流程如下：
1. 从memdb中读取
2. 从frozen memdb中读取
3. 从SST File列表中读取
  

遍历各Level详细流程
1. 从0层文件列表中读取，按照文件描述符降序读取。全部找完都没有再去非0层文件列表中读取。
2. 从非0层文件列表中读取，根据二分定位可能存在的File。本level此文件找完找不到数据，则去下一层。
3. 读流程结束
![读概览.png](/pic/leveldb/read_section.png)   
![读概览.png](/pic/leveldb/read_from_level.png)  

如下是从当前文件中找key流程
1. 读取footer中各index block索引
2. 读取各Index Block数据
3. 根据IndexBlock定位 DataBlock
4. 根据DataBlock定位FilterBlock
5. 读取DataBlock数据遍历找key
![读概览.png](/pic/leveldb/find_key_from_file.png)  

从文件中找key流程如下
```go
// NOTE 从文件中找最近一个比本Key大的Key并返回值
func (r *Reader) find(key []byte, filtered bool, ro *opt.ReadOptions, noValue bool) (rkey, value []byte, err error) {
    ...
    // NOTE 获取当前文件的indexBlock
    indexBlock, rel, err := r.getIndexBlock(true)
    if err != nil {
       return
    }
    defer rel.Release()

    index := r.newBlockIter(indexBlock, nil, nil, true)
    defer index.Release()
    // NOTE 从indexBlock中定位出这个key应该在的Block(2分法定位，大于等于位置)
    if !index.Seek(key) {
       if err = index.Error(); err == nil {
          err = ErrNotFound
       }
       return
    }
    // NOTE 取出indexBlock指定位置的val——DataBlock的索引
    dataBH, n := decodeBlockHandle(index.Value())
    if n == 0 {
       r.err = r.newErrCorruptedBH(r.indexBH, "bad data block handle")
       return nil, nil, r.err
    }
    // NOTE 如果有过滤器则判断
    // 这里过滤器生成以及判断DataBlock offset对应哪个Filter的逻辑比较绕
    // The filter should only used for exact match.
    if filtered && r.filter != nil {
       filterBlock, frel, ferr := r.getFilterBlock(true)
       if ferr == nil {
          if !filterBlock.contains(r.filter, dataBH.offset, key) { // 拿dataBlock Offset去找Filter比较绕
             frel.Release()
             return nil, nil, ErrNotFound // filter没有就真没有
          }
          frel.Release()
       } else if !errors.IsCorrupted(ferr) {
          return nil, nil, ferr
       }
    }
   
    data := r.getDataIter(dataBH, nil, r.verifyChecksum, !ro.GetDontFillCache())
    if !data.Seek(key) { // 没有找到比本key大的，在本文件中，刚好比本key大的要么成功找到，要么此key不存在本文件中
       data.Release()
       if err = data.Error(); err != nil {
          return
       }
       ...
    }
    ...
    return
}
```
>问题：为什么能够根据DataBlock的Offset找到对应filter？BlockSize是4KB，而filter是2KB创建一个，但DataBlock和Filter是多对一的关系，如何查找？   
> 答：filter及其offset按照如下图方式生成；详细解析见 https://www.huliujia.com/blog/1a8dbd0991/

### Compaction流程
背景：随着写入的sstable越来越多，查询也会越来越慢，leveldb中使用compaction减少文件的个数，加快查询的速度、降低空间使用量。  

Compaction分为两种：Minor Compaction 和 Major Compaction  

#### MinorCompaction
场景：Frozen memdb 落成 SST 文件
1. 遍历memdb写入buffer中；
2. buffer满则写入restart数组并将此DataBlock写入文件；
3. 生成Filter。重复1~3流程值写入完成；
4. 写filterBlock、Meta IndexBlock、IndexBlock；
5. 写入footer；

![minor.png](/pic/leveldb/frozen_memdb_compaction.png)  


#### MajorCompaction
场景：SST文件之间进行Compaction；  

实现：选取一个文件集合(合并后不与本次其他文件范围重叠)，每一个文件创建一个iterator，比较大小后写入。  

触发时机：
1. Manual Compaction：人工调接口触发
2. Size Compaction：是根据每个level的总文件大小、文件数量超过阈值来触发。
3. Seek Compaction：每个文件的 seek miss 次数都有一个阈值，如果超过了这个阈值，那么认为这个文件需要Compact。
1. 备注：针对不同文件大小，seek压缩的阈值不同；一般认为seek次数过多代表压缩能够换来更快的性能。

0层文件数量、非0层文件大小 阈值判断是否需要compaction函数逻辑如下  
```go
func (v *version) computeCompaction() {
    ...
    // 遍历每层数据
    for level, tables := range v.levels {
       var score float64
       size := tables.size()
       if level == 0 { // NOTE 文件数量比例是否超过阈值
          // We treat level-0 specially by bounding the number of files
          // instead of number of bytes for two reasons:
          //
          // (1) With larger write-buffer sizes, it is nice not to do too
          // many level-0 compaction.
          //
          // (2) The files in level-0 are merged on every read and
          // therefore we wish to avoid too many files when the individual
          // file size is small (perhaps because of a small write-buffer
          // setting, or very high compression ratios, or lots of
          // overwrites/deletions).
          score = float64(len(tables)) / float64(v.s.o.GetCompactionL0Trigger())
       } else { // 其他层文件大小总和是否超过阈值
          score = float64(size) / float64(v.s.o.GetCompactionTotalSize(level))
       }

       if score > bestScore {
          bestLevel = level
          bestScore = score // score>1，该层就需要压缩
       }

       statFiles[level] = len(tables)
       statSizes[level] = shortenb(size)
       statScore[level] = fmt.Sprintf("%.2f", score)
       statTotSize += size
    }

    v.cLevel = bestLevel
    v.cScore = bestScore

    v.s.logf("version@stat F·%v S·%s%v Sc·%v", statFiles, shortenb(statTotSize), statSizes, statScore)
}

// 获取每一层文件大小阈值
func (o *Options) GetCompactionTotalSize(level int) int64 {
    var (
       base = DefaultCompactionTotalSize
       mult float64
    )
    if o != nil {
       if o.CompactionTotalSize > 0 {
          base = o.CompactionTotalSize
       }
       if level < len(o.CompactionTotalSizeMultiplierPerLevel) && o.CompactionTotalSizeMultiplierPerLevel[level] > 0 {
          mult = o.CompactionTotalSizeMultiplierPerLevel[level]
       } else if o.CompactionTotalSizeMultiplier > 0 {
          mult = math.Pow(o.CompactionTotalSizeMultiplier, float64(level))
       }
    }
    if mult == 0 {
       mult = math.Pow(DefaultCompactionTotalSizeMultiplier, float64(level))
    }
    return int64(float64(base) * mult)
}
```
查找次数判断  
```go
// tFile holds basic information about a table.
type tFile struct {
    fd         storage.FileDesc
    seekLeft   int32
    size       int64
    imin, imax internalKey
}
```
### 事务写(略)
事务写流程如下，主要是
1. 拿到写锁，执行前将memdb变成frozen memdb
2. 创建一个新的mpool用来存储后续事务中的写数据；
3. 事务中的写都往mpool中写，提交时将mpool落成一个SST文件
4. 将提交新增本SST文件的record——其他人可见

## 其他
存储空间优化——变长int存储  

>使用目的：降低存储空间使用  
> 
>变长int存储原理：每八位中最高位标识下一个Byte是否有效，剩余7位标识真实数字。
```go
// PutUvarint encodes a uint64 into buf and returns the number of bytes written.
// If the buffer is too small, PutUvarint will panic.
func PutUvarint(buf []byte, x uint64) int {
    i := 0
    for x >= 0x80 {
       buf[i] = byte(x) | 0x80
       x >>= 7
       i++
    }
    buf[i] = byte(x)
    return i + 1
}

// Uvarint decodes a uint64 from buf and returns that value and the
// number of bytes read (> 0). If an error occurred, the value is 0
// and the number of bytes n is <= 0 meaning:
//
//  n == 0: buf too small
//  n  < 0: value larger than 64 bits (overflow)
//          and -n is the number of bytes read
func Uvarint(buf []byte) (uint64, int) {
    var x uint64
    var s uint
    for i, b := range buf {
       if i == MaxVarintLen64 {
          // Catch byte reads past MaxVarintLen64.
          // See issue https://golang.org/issues/41185
          return 0, -(i + 1) // overflow
       }
       if b < 0x80 {
          if i == MaxVarintLen64-1 && b > 1 {
             return 0, -(i + 1) // overflow
          }
          return x | uint64(b)<<s, i + 1
       }
       x |= uint64(b&0x7f) << s
       s += 7
    }
    return 0, 0
}
```

实例宕机恢复流程(略)  

过程：
1. 数据库启动读取current文件，获取当前生效manifest文件名
2. 获取最近一个有效的version
3. 向此version应用record以恢复系统状态
4. 从WAL恢复memdb

LRU Cache(略)  

过程：
1. 双向链表+HashMap实现LRU Cache
2. 会将打开文件的Reader、Index Block、MetaIndex Block、FilterBlock、部分DataBlock放入其中以加速读取；

## 扩展
> LSM思想、LevelDB应用的其他场景

分布式布隆过滤器:后台流程定时将不可变memtable 合并进bloom文件，然后在线实时追踪可变memtable，异步监听bloom文件的变更。  

RocksDB  

leveldb和rocksdb有什么区别？  

多线程压缩、锁优化、前缀bloom filter、memtable bloom filter、Column Families。详见：https://github.com/facebook/rocksdb/wiki/Features-Not-in-LevelDB  

RocksDB代码仓：https://github.com/facebook/rocksdb

## 参考文档
>按顺序代表推荐优先级

本文代码讲述以github上开源的goleveldb为基础：https://github.com/syndtr/goleveldb

goleveldb手册:https://leveldb-handbook.readthedocs.io/zh/latest/sstable.html#index-block (推荐阅读)  

B站陈硕：LevelDB磁盘数据结构解析 https://www.bilibili.com/video/BV1R84y1w77r/ (推荐阅读)

LSM论文：https://www.cs.umb.edu/~poneil/lsmtree.pdf
