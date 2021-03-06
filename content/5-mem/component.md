# 5 内存管理: 分配器组件

本节循序渐进地独立讨论内存分配器中的几个组件：`fixalloc`、`linearAlloc`、`mcache`。

## fixalloc

`fixalloc` 是一个基于自由列表的固定大小的分配器。其核心原理是将若干未分配的内存块连接起来，
将未分配的区域的第一个字为指向下一个未分配区域的指针使用。

Go 的主分配堆中 malloc（span、cache、treap、finalizer、profile、arena hint 等） 均
围绕它为实体进行固定分配和回收。

fixalloc 作为抽象，非常简洁，只包含三个基本操作：初始化、分配、回收

### 结构

```go
// fixalloc 是一个简单的固定大小对象的自由表内存分配器。
// Malloc 使用围绕 sysAlloc 的 fixalloc 来管理其 MCache 和 MSpan 对象。
//
// fixalloc.alloc 返回的内存默认为零，但调用者可以通过将 zero 标志设置为 false
// 来自行负责将分配归零。如果这部分内存永远不包含堆指针，则这样的操作是安全的。
//
// 调用方负责锁定 fixalloc 调用。调用方可以在对象中保持状态，
// 但当释放和重新分配时第一个字会被破坏。
//
// 考虑使 fixalloc 的类型变为 go:notinheap.
type fixalloc struct {
	size   uintptr
	first  func(arg, p unsafe.Pointer) // 首次调用时返回 p
	arg    unsafe.Pointer
	list   *mlink
	chunk  uintptr // 使用 uintptr 而非 unsafe.Pointer 来避免 write barrier
	nchunk uint32
	inuse  uintptr // 正在使用的字节
	stat   *uint64
	zero   bool // 归零的分配
}
```

### 初始化

Go 语言对于零值有自己的规定，自然也就体现在内存分配器上。而 `fixalloc` 作为内存分配器内部组件的来源于
操作系统的内存，自然需要自行初始化，因此，`fixalloc` 的初始化也就不可避免的需要将自身的各个字段归零：

```go
// 初始化 f 来分配给定大小的对象。
// 使用分配器来按 chunk 获取
func (f *fixalloc) init(size uintptr, first func(arg, p unsafe.Pointer), arg unsafe.Pointer, stat *uint64) {
	f.size = size
	f.first = first
	f.arg = arg
	f.list = nil
	f.chunk = 0
	f.nchunk = 0
	f.inuse = 0
	f.stat = stat
	f.zero = true
}
```

### 分配

`fixalloc` 基于自由表策略进行实现，分为两种情况：

1. 存在被释放、可复用的内存
2. 不存在可复用的内存

对于第一种情况，也就是在运行时内存被释放，但这部分内存并不会被立即回收给操作系统，
我们直接从自由表中获得即可，但需要注意按需将这部分内存进行清零操作。

对于第二种情况，我们直接向操作系统申请固定大小的内存，然后扣除分配的大小即可。

```go
const 	_FixAllocChunk = 16 << 10               // FixAlloc 一个 Chunk 的大小

func (f *fixalloc) alloc() unsafe.Pointer {
	// fixalloc 的个字段必须先被 init
	if f.size == 0 {
		print("runtime: use of FixAlloc_Alloc before FixAlloc_Init\n")
		throw("runtime: internal error")
	}

	// 如果 f.list 不是 nil, 则说明还存在已经释放、可复用的内存，直接将其分配
	if f.list != nil {
		// 取出 f.list
		v := unsafe.Pointer(f.list)
		// 并将其指向下一段区域
		f.list = f.list.next
		// 增加使用的(分配)大小
		f.inuse += f.size
		// 如果需要对内存清零，则对取出的内存执行初始化
		if f.zero {
			memclrNoHeapPointers(v, f.size)
		}
		// 返回分配的内存
		return v
	}

	// f.list 中没有可复用的内存

	// 如果此时 nchunk 不足以分配一个 size
	if uintptr(f.nchunk) < f.size {
		// 则向操作系统申请内存，大小为 16 << 10 pow(2,14)
		f.chunk = uintptr(persistentalloc(_FixAllocChunk, 0, f.stat))
		f.nchunk = _FixAllocChunk
	}

	// 指向申请好的内存
	v := unsafe.Pointer(f.chunk)
	if f.first != nil { // first 只有在 fixalloc 作为 spanalloc 时候，才会被设置为 recordspan
		f.first(f.arg, v) // 用于为 heap.allspans 添加新的 span
	}
	// 扣除并保留 size 大小的空间
	f.chunk = f.chunk + f.size
	f.nchunk -= uint32(f.size)
	f.inuse += f.size // 记录已经使用的大小
	return v
}
```

上面的代码中：

- `memclrNoHeapPointers` 具体实现分析见 [4 调度器: 初始化](../4-sched/init.md)。
- `persistentalloc` 具体实现分析见 [5 内存分配器: 全局分配](../5-mem/galloc.md)

### 回收

回收就更加简单了，直接将回收的地址指针放回到自由表中即可：

```go
func (f *fixalloc) free(p unsafe.Pointer) {
	// 减少使用的字节数
	f.inuse -= f.size
	// 将要释放的内存地址作为 mlink 指针插入到 f.list 内，完成回收
	v := (*mlink)(p)
	v.next = f.list
	f.list = v
}
```

## linearAlloc

`linearAlloc` 是一个基于线性分配策略的分配器，但由于它只作为 `mheap_.heapArenaAlloc` 和 `mheap_.arena`
在 32 位系统上使用，这里不做详细分析。

```go
// linearAlloc 是一个简单的线性分配器，提前储备了内存的一块区域并在需要时指向该区域。
// 调用方负责加锁。
type linearAlloc struct {
	next   uintptr // 下一个可用的字节
	mapped uintptr // 映射空间后的一个字节
	end    uintptr // 保留空间的末尾
}

func (l *linearAlloc) init(base, size uintptr) {
	l.next, l.mapped = base, base
	l.end = base + size
}

func (l *linearAlloc) alloc(size, align uintptr, sysStat *uint64) unsafe.Pointer {
	p := round(l.next, align)
	if p+size > l.end {
		return nil
	}
	l.next = p + size
	if pEnd := round(l.next-1, physPageSize); pEnd > l.mapped {
		// We need to map more of the reserved space.
		sysMap(unsafe.Pointer(l.mapped), pEnd-l.mapped, sysStat)
		l.mapped = pEnd
	}
	return unsafe.Pointer(p)
}
```

## mcache

`mcache` 是一个 per-P 的缓存，因此每个线程都只访问自身的 `mcache`，因此也就不会出现
并发，也就省去了对其进行加锁步骤。

```go
//go:notinheap
type mcache struct {
	// 下面的成员在每次 malloc 时都会被访问
	// 因此将它们放到一起来利用缓存的局部性原理
	next_sample int32   // 分配这么多字节后触发堆样本
	local_scan  uintptr // 分配的可扫描堆的字节数

	// 没有指针的微小对象的分配器缓存。
	// 请参考 malloc.go 中的 "小型分配器" 注释。
	//
	// tiny 指向当前 tiny 块的起始位置，或当没有 tiny 块时候为 nil
	// tiny 是一个堆指针。由于 mcache 在非 GC 内存中，我们通过在
	// mark termination 期间在 releaseAll 中清除它来处理它。
	tiny             uintptr
	tinyoffset       uintptr
	local_tinyallocs uintptr // 不计入其他统计的极小分配的数量

	// 下面的不在每个 malloc 时被访问

	alloc [numSpanClasses]*mspan // 用来分配的 spans，由 spanClass 索引

	stackcache [_NumStackOrders]stackfreelist

	// 本地分配器统计，在 GC 期间被刷新
	local_largefree  uintptr                  // bytes freed for large objects (>maxsmallsize)
	local_nlargefree uintptr                  // number of frees for large objects (>maxsmallsize)
	local_nsmallfree [_NumSizeClasses]uintptr // number of frees for small objects (<=maxsmallsize)

	// flushGen indicates the sweepgen during which this mcache
	// was last flushed. If flushGen != mheap_.sweepgen, the spans
	// in this mcache are stale and need to the flushed so they
	// can be swept. This is done in acquirep.
	flushGen uint32
}
```

### 分配

运行时的 `runtime.allocmcache` 从 `mheap` 上分配一个 `mcache`。
由于 `mheap` 是全局的，因此在分配期必须对其进行加锁，而分配通过 fixAlloc 组件完成：

```go
// 虚拟的MSpan，不包含任何对象。
var emptymspan mspan

func allocmcache() *mcache {
	lock(&mheap_.lock)
	c := (*mcache)(mheap_.cachealloc.alloc())
	c.flushGen = mheap_.sweepgen
	unlock(&mheap_.lock)
	for i := range c.alloc {
		c.alloc[i] = &emptymspan // 暂时指向虚拟的 mspan 中
	}
	// 返回下一个采样点，是服从泊松过程的随机数
	c.next_sample = nextSample()
	return c
}
```

由于运行时提供了采样过程堆分析的支持，
由于我们的采样的目标是平均每个 `MemProfileRate` 字节对分配进行采样，
显然，在整个时间线上的分配情况应该是完全随机分布的，这是一个泊松过程。
因此最佳的采样点应该是服从指数分布 `exp(MemProfileRate)` 的随机数，其中
`MemProfileRate` 为均值。

```go
func nextSample() int32 {
	if GOOS == "plan9" {
		// Plan 9 doesn't support floating point in note handler.
		if g := getg(); g == g.m.gsignal {
			return nextSampleNoFP()
		}
	}

	return fastexprand(MemProfileRate) // 即 exp(MemProfileRate)
}
```

`MemProfileRate` 是一个公共变量，可以在用户态代码进行修改：

```go
var MemProfileRate int = 512 * 1024
```

### 释放

由于 `mcache` 从非 GC 内存上进行分配，因此出现的任何堆指针都必须进行特殊处理。
所以在释放前，需要调用 `mcache.releaseAll` 将堆指针进行处理：

```go
func (c *mcache) releaseAll() {
	for i := range c.alloc {
		s := c.alloc[i]
		if s != &emptymspan {
			// 将 span 归还
			mheap_.central[i].mcentral.uncacheSpan(s)
			c.alloc[i] = &emptymspan
		}
	}
	// 清空 tinyalloc 池.
	c.tiny = 0
	c.tinyoffset = 0
}
```

```go
func freemcache(c *mcache) {
	systemstack(func() {
		// 归还 span
		c.releaseAll()
		// 释放 stack
		stackcache_clear(c)

		lock(&mheap_.lock)
		// 记录局部统计
		purgecachedstats(c)
		// 将 mcache 释放
		mheap_.cachealloc.free(unsafe.Pointer(c))
		unlock(&mheap_.lock)
	})
}
```

### per-P? per-M?

mcache 其实早在 [4 调度器: 调度循环](../4-sched/exec.md) 中与 mcache 打过照面了。

首先，mcache 是一个 per-P 的 mcache，我们很自然的疑问就是，这个 mcache 在 p/m 这两个结构体上都有成员：

```go
type p struct {
	(...)
	mcache      *mcache
	(...)
}
type m struct {
	(...)
	mcache      *mcache
	(...)
}
```

那么 mcache 是跟着谁跑的？结合调度器的知识不难发现，m 在执行时需要持有一个 p 才具备执行能力。
有利的证据是，当调用 `runtime.procresize` 时，初始化新的 P 时，mcache 是直接分配到 p 的；
回收 p 时，mcache 是直接从 p 上获取：

```go
func procresize(nprocs int32) *p {
	(...)
	// 初始化新的 P
	for i := int32(0); i < nprocs; i++ {
		pp := allp[i]
		(...)
		// 为 P 分配 cache 对象
		if pp.mcache == nil {
			if old == 0 && i == 0 {
				if getg().m.mcache == nil {
					throw("missing mcache?")
				}
				pp.mcache = getg().m.mcache
			} else {
				// 创建 cache
				pp.mcache = allocmcache()
			}
		}

		(...)
	}

	// 释放未使用的 P
	for i := nprocs; i < old; i++ {
		p := allp[i]
		(...)
		// 释放当前 P 绑定的 cache
		freemcache(p.mcache)
		p.mcache = nil
		(...)
	}
	(...)
}
```

因而我们可以明确：

- mcache 会被 P 持有，当 M 和 P 绑定时，M 同样会保留 mcache 的指针
- mcache 直接向操作系统申请内存，且常驻运行时
- P 通过 make 命令进行分配，会分配在 Go 堆上

## 许可

[Go under the hood](https://github.com/changkun/go-under-the-hood) | CC-BY-NC-ND 4.0 & MIT &copy; [changkun](https://changkun.de)
