# bigcache
快速，并发，退出内存中的高速缓存被写入，以保留大量条目，而不会影响性能。 BigCache将条目保留在堆上，但省略了它们的GC。 为此，需要对字节片进行操作，因此在大多数使用情况下，需要在缓存前面进行条目（反序列化）。

Requires Go 1.12 or newer.

原理
-------
bigcache几个核心的数据结构：
* liftWindown
* hasher
* cacheShard :sync.RWMutex 、map、queue.BytesQueue
* shardMask
* Config

##### hasher:
通过hasher提供的hash功能将字符串类型的key转换为uint64类型的hash值。该值有两个用途：
1. 作为[]*cacheShard切片的索引，获取该key对应的cacheShard指针
2. 作为cacheShard类型中map[uint64]uint32类型字段的key
因为对bigcache结构的相关操作是无锁操作，所以分片技术是并发性能提升的关键

##### cacheShard：
单个分片类型，缓存的核心功能由该类型提供，由sync.RWMutex字段可以得知。他是多线程安全的，其中map[uint64]uint32和queue.BytesQueue是零GC的关键。

##### queue.BytesQueue：
queue是该包的一个子包，BytesQueue是一个基于字节数组的FIFO队列。某个分片中所有的缓存值都存放在这个字节数组中。当一个记录加入缓存时，它的值会被push到该队列的尾部，同时返回记录的值在该字节数组中的索引，这个索引就是cacheShard类型中map[uint64]uint32字段的value。BytesQueue使用了变长的整数，这是通过标准库encoding/binary实现的
##### leftWindow:
多久后自己自动删除。bigcache的淘汰策略使用的是时间策略，即根据固定的一个过期时间用FIFO算法进行淘汰

##### 并发优化：
数据分片
##### GC优化：
把hash值作为map[uint64]uint32的key,把缓存对象序列化后放到一个预先分配的大的字节数组中，然后把它在数组中的offset作为map[uin64]uint32的value。
使用

-------
#### 简单初始化：
```
import "github.com/allegro/bigcache"

cache, _ := bigcache.NewBigCache(bigcache.DefaultConfig(10 * time.Minute))

cache.Set("my-unique-key", []byte("value"))

entry, _ := cache.Get("my-unique-key")
fmt.Println(string(entry))
```
#### 自定义初始化：
如果可以提前预测缓存负载，则最好使用自定义初始化，因为这样可以避免额外的内存分配。
```
import (
	"log"

	"github.com/allegro/bigcache"
)

config := bigcache.Config {
		// 分片的数量 (必须是2的幂)
		Shards: 1024,

		// 剩余时间窗口time after which entry can be evicted
		LifeWindow: 10 * time.Minute,

		// 清除过期记录Interval between removing expired entries (清理).
		// 如果设置为 <= 0 则不会执行任何操作
		// bigcache的时间单位是秒，设置为 < 1秒 则不会起作用		
		CleanWindow: 5 * time.Minute,

		// rps * lifeWindow, 仅用于初始内存分配
		MaxEntriesInWindow: 1000 * 10 * 60,

		// 最大记录数的字节数,仅仅用于初始化内存分配
		MaxEntrySize: 500,

		// 打印额外的内存分配信息
		Verbose: true,

		// 缓存大小不能超过这个限制值，值的单位是 MB
		// 如果达到了最大限制值，最老的记录会被最新的记录覆盖
		// 如果设置为0 ，就是没有大小限制
		HardMaxCacheSize: 8192,

		// 当最老的记录因为过期被移除或内存剩余空间不足或者因为调用了delete方法的时候，会触发回调方法，并且返回代表原因的位掩码
		// 默认值是nil，即没有回调方法and it prevents from unwrapping the oldest entry.
		OnRemove: nil,

		// OnRemoveWithReason 是回调方法，当最老的记录因为过期被移除或内存剩余空间不足或者因为调用了delete方法的时候，会触发该回调方法，并且传递代表原因的常量值 
		// Default value is nil which means no callback and it prevents from unwrapping the oldest entry.
		// 如果指定了OnRemove，则忽略
		OnRemoveWithReason: nil,
	}

cache, initErr := bigcache.NewBigCache(config)
if initErr != nil {
	log.Fatal(initErr)
}

cache.Set("my-unique-key", []byte("value"))

if entry, err := cache.Get("my-unique-key"); err == nil {
	fmt.Println(string(entry))
}
```

LifeWindow & CleanWindow
-------
1. LifeWindow是一个时间点。 在此之后，条目可以称为无效条目，但不能删除。
2. CleanWindow是一个时间点。 在那之后，所有无效条目将被删除，但仍然有效的条目将不被删除。

基准测试
-------
Three caches were compared: bigcache, freecache and map. Benchmark tests were made using an i7-6700K CPU @ 4.00GHz with 32GB of RAM on Ubuntu 18.04 LTS (5.2.12-050212-generic).

Benchmarks source code can be found here

##### 读和写：

```
go version
go version go1.13 linux/amd64

go test -bench=. -benchmem -benchtime=4s ./... -timeout 30m
goos: linux
goarch: amd64
pkg: github.com/allegro/bigcache/v2/caches_bench
BenchmarkMapSet-8                     	12999889	       376 ns/op	     199 B/op	       3 allocs/op
BenchmarkConcurrentMapSet-8           	 4355726	      1275 ns/op	     337 B/op	       8 allocs/op
BenchmarkFreeCacheSet-8               	11068976	       703 ns/op	     328 B/op	       2 allocs/op
BenchmarkBigCacheSet-8                	10183717	       478 ns/op	     304 B/op	       2 allocs/op
BenchmarkMapGet-8                     	16536015	       324 ns/op	      23 B/op	       1 allocs/op
BenchmarkConcurrentMapGet-8           	13165708	       401 ns/op	      24 B/op	       2 allocs/op
BenchmarkFreeCacheGet-8               	10137682	       690 ns/op	     136 B/op	       2 allocs/op
BenchmarkBigCacheGet-8                	11423854	       450 ns/op	     152 B/op	       4 allocs/op
BenchmarkBigCacheSetParallel-8        	34233472	       148 ns/op	     317 B/op	       3 allocs/op
BenchmarkFreeCacheSetParallel-8       	34222654	       268 ns/op	     350 B/op	       3 allocs/op
BenchmarkConcurrentMapSetParallel-8   	19635688	       240 ns/op	     200 B/op	       6 allocs/op
BenchmarkBigCacheGetParallel-8        	60547064	        86.1 ns/op	     152 B/op	       4 allocs/op
BenchmarkFreeCacheGetParallel-8       	50701280	       147 ns/op	     136 B/op	       3 allocs/op
BenchmarkConcurrentMapGetParallel-8   	27353288	       175 ns/op	      24 B/op	       2 allocs/op
PASS
ok  	github.com/allegro/bigcache/v2/caches_bench	256.257s
```
bigcache中的读写比freecache中的更快。 写入map最慢。

GC暂停时间
-------

```
go version
go version go1.13 linux/amd64

go run caches_gc_overhead_comparison.go

Number of entries:  20000000
GC pause for bigcache:  1.506077ms
GC pause for freecache:  5.594416ms
GC pause for map:  9.347015ms
```

```
go version
go version go1.13 linux/arm64

go run caches_gc_overhead_comparison.go
Number of entries:  20000000
GC pause for bigcache:  22.382827ms
GC pause for freecache:  41.264651ms
GC pause for map:  72.236853ms
```
测试显示，在处理2000万记录缓存的是偶GC暂停的时间。 Bigcache和freecache的GC暂停时间非常相似。

内存使用情况
-------
您可能会遇到系统内存报告似乎呈指数增长的情况，但是这是预期的行为。 Go运行时以块或“跨度”分配内存，并通过将其状态更改为“空闲”来通知OS何时不再需要它们。 “跨度”将一直是进程资源使用的一部分，直到OS需要重新分配地址用途为止。
[Go basically never frees heap memory back to the operating system](https://utcc.utoronto.ca/~cks/space/blog/programming/GoNoMemoryFreeing)

如何工作的
-------
BigCache依赖于Go（issue 9477）1.5版中提供的优化。 此优化表明，如果使用在键和值中没有指针的映射，则GC将忽略其内容。 因此，BigCache使用map [uint64] uint32，其中键是哈希值，值是条目的偏移量。

条目保留在字节片中，以再次省略GC。 字节片大小可以增长到千兆字节，而不会影响性能，因为GC仅会看到指向它的单个指针。

hash冲突
-------
BigCache不处理冲突。 当插入新项目并且其哈希与先前存储的项目冲突时，新项目将覆盖先前存储的值。

Bigcache与Freecache对比
-------
两种缓存都提供相同的核心功能，但是它们以不同的方式减少了GC开销。 Bigcache依赖于map [uint64] uint32，freecache实现了自己的基于切片的映射，以减少指针数量。

基准测试的结果如上所示。 与免费缓存相比，bigcache的优点之一是您不需要预先知道缓存的大小，因为bigcache满了时，它可以为新条目分配额外的内存，而不必像freecache那样覆盖现有条目。 但是也可以设置bigcache中的硬最大大小，请检查HardMaxCacheSize。

HTTP Server
-------
该软件包还包括BigCache的易于部署的HTTP实现：[server](https://github.com/allegro/bigcache/tree/master/server)

More
-------
在allegro.tech博客文章中介绍了Bigcache的起源：[writing a very fast cache service in Go](https://allegro.tech/2016/03/writing-fast-cache-service-in-go.html)