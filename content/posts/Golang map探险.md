---
title: "Golang map探险"
date: 2020-12-05T23:06:41+08:00
draft: false
original: true
mathjax: true
categories: 
  - Golang
tags: 
  - Golang基础
---

### 前言

map是一种很基础的数据结构，Java中的HashMap使用数组+链表的方式来实现，Golang内置map底层是什么样的结构呢？

### 内存结构

在go中map使用的是`hmap`这个结构来表示

```go
// A header for a Go map.
type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
	count     int // map 中的元素个数，必须放在 struct 的第一个位置，因为 内置的 len 函数会从这里读取
	flags     uint8
	B         uint8  // log_2 of # of buckets (最多可以放 loadFactor * 2^B 个元素，再多就要 hashGrow 了)
	noverflow uint16 // overflow 的 bucket 的近似数
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // 2^B 大小的数组，如果 count == 0 的话，可能是 nil
	oldbuckets unsafe.Pointer // 一半大小的之前的 bucket 数组，只有在 growing 过程中是非 nil
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

	extra *mapextra // 当 key 和 value 都可以 inline 的时候，就会用这个字段
}
```

<!--more-->

![hmap结构](/Golang-map探险/hmap.png)

其中`buckets`字段指向bucket数组结构，这个bucket数组的长度是2^B

bucket是用`bmap`这个结构来表示的

```go
const (
  // Maximum number of key/elem pairs a bucket can hold.
  bucketCntBits = 3
  bucketCnt     = 1 << bucketCntBits
)

// A bucket for a Go map.
type bmap struct {
	// tophash generally contains the top byte of the hash value
	// for each key in this bucket. If tophash[0] < minTopHash,
	// tophash[0] is a bucket evacuation state instead.
	tophash [bucketCnt]uint8
	// Followed by bucketCnt keys and then bucketCnt elems.
	// NOTE: packing all the keys together and then all the elems together makes the
	// code a bit more complicated than alternating key/elem/key/elem/... but it allows
	// us to eliminate padding which would be needed for, e.g., map[int64]int8.
	// Followed by an overflow pointer.
}

type mapextra struct {
    // 如果 key 和 value 都不包含指针，并且可以被 inline(<=128 字节)
    // 使用 extra 来存储 overflow bucket，这样可以避免 GC 扫描整个 map
    // 然而 bmap.overflow 也是个指针。这时候我们只能把这些 overflow 的指针
    // 都放在 hmap.extra.overflow 和 hmap.extra.oldoverflow 中了
    // overflow 包含的是 hmap.buckets 的 overflow 的 bucket
    // oldoverflow 包含扩容时的 hmap.oldbuckets 的 overflow 的 bucket
    overflow    *[]*bmap
    oldoverflow *[]*bmap

    // 指向空闲的 overflow bucket 的指针
    nextOverflow *bmap
}
```

然而事情并没有这么简单，由于map可以存储不同类型的键值，目前go也不支持泛型，所以键值占据的内存空间大小只能在编译时期进行推导，编译期间`cmd/compile/internal/gc/reflect.go#bmap`会给动态地创建一个新的结构：

```go
type bmap struct {
    topbits  [8]uint8
    keys     [8]keytype 
    values   [8]valuetype
    pad      uintptr
    overflow uintptr
}
```

![bmap结构](/Golang-map探险/bmap.png)

可以看到每个桶存储的键值对的个数是固定的8个，而且key和value的形式并不像Java中使用Map.Entry<K,V>这种方式存储，而是8个key放在一起，8个value放在一起，这种方式的好处是，只需要在最后进行补齐，不用为每个Map.Entry<K,V>进行补齐

如果bucket中8个位置都满了，就会创建新的bucket，并用overflow指向这个新的bucket

![bmap链](/Golang-map探险/bmap链.png)

目前为止，我们了解到的map就是这个样子的

![bmap结构](/Golang-map探险/hmap整体.png)

### 创建map

平时我们创建map有很多种方式：

字面量的方式创建
```go
hash := map[string]int{
	"1": 2,
	"3": 4,
	"5": 6,
}
```

使用字面量的方式初始化map，最终都会通过`cmd/compile/internal/gc.maplit`函数初始化，详情可参见[字面量初始化map](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap/#%E5%AD%97%E9%9D%A2%E9%87%8F)

使用make不指定大小的方式
```go
hash := make(map[string]int)
```

使用make指定大小的方式
```go
hash := make(map[string]int, 8)
```

在运行时使用make初始化map，这种方式由编译器来确定如何初始化，入口在`cmd/compile/internal/gc/walk.go`中`case OMAKEMAP:`分支，会根据是否逃逸，堆上还是栈上，初始化大小长度与8的关系等使用不同的初始化逻辑，其中最重要的就是`makemap`


```go
// makemap implements Go map creation for make(map[k]v, hint).
// If the compiler has determined that the map or the first bucket
// can be created on the stack, h and/or bucket may be non-nil.
// If h != nil, the map can be created directly in h.
// If h.buckets != nil, bucket pointed to can be used as the first bucket.
func makemap(t *maptype, hint int, h *hmap) *hmap {
	mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
	if overflow || mem > maxAlloc {
		hint = 0
	}

	// initialize Hmap
	if h == nil {
		h = new(hmap)
	}
	h.hash0 = fastrand()

	// Find the size parameter B which will hold the requested # of elements.
	// For hint < 0 overLoadFactor returns false since hint < bucketCnt.
	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B

	// allocate initial hash table
	// if B == 0, the buckets field is allocated lazily later (in mapassign)
	// If hint is large zeroing this memory could take a while.
	if h.B != 0 {
		var nextOverflow *bmap
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}

	return h
}
```

这个函数的执行过程会分成以下几个部分：

1. 计算哈希占用的内存是否溢出或者超出能分配的最大值；
2. 调用 fastrand 获取一个随机的哈希种子；
3. 根据传入的 hint 计算出需要的最小需要的桶的数量；
4. 使用 runtime.makeBucketArray 创建用于保存桶的数组；

`runtime.makeBucketArray` 函数会根据传入的 B 计算出的需要创建的桶数量在内存中分配一片连续的空间用于存储数据:

```go
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
	base := bucketShift(b)
	nbuckets := base
	if b >= 4 {
		nbuckets += bucketShift(b - 4)
		sz := t.bucket.size * nbuckets
		up := roundupsize(sz)
		if up != sz {
			nbuckets = up / t.bucket.size
		}
	}

	buckets = newarray(t.bucket, int(nbuckets))
	if base != nbuckets {
		nextOverflow = (*bmap)(add(buckets, base*uintptr(t.bucketsize)))
		last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.bucketsize)))
		last.setoverflow(t, (*bmap)(buckets))
	}
	return buckets, nextOverflow
}
```

* 当桶的数量小于 $2^4$ 时，由于数据较少、使用溢出桶的可能性较低，这时就会省略创建的过程以减少额外开销；
* 当桶的数量多于 $2^4$ 时，就会额外创建 $2^{𝐵−4}$ 个溢出桶；

根据上述代码，我们能确定在正常情况下，正常桶和溢出桶在内存中的存储空间是连续的，只是被`hmap`中的不同字段引用，当溢出桶数量较多时会通过`runtime.newobject`创建新的溢出桶。

### 访问map

平时我们使用map有2种方式

1. 通过key

```go
v := hash[key]
```

在编译的类型检查期间，hash[key] 以及类似的操作都会被转换成对哈希的 OINDEXMAP 操作，中间代码生成阶段会在 cmd/compile/internal/gc.walkexpr 函数中将这些 OINDEXMAP 操作转换成如下的代码：

```
v     := hash[key] // => v     := *mapaccess1(maptype, hash, &key)
v, ok := hash[key] // => v, ok := mapaccess2(maptype, hash, &key)
```

赋值语句左侧接受参数的个数会决定使用的运行时方法：

1. 当接受参数仅为一个时，会使用 runtime.mapaccess1，该函数仅会返回一个指向目标值的指针；
2. 当接受两个参数时，会使用 runtime.mapaccess2，除了返回目标值之外，它还会返回一个用于表示当前键对应的值是否存在的布尔值

`runtime.mapaccess1` 函数会先通过哈希表设置的哈希函数、种子获取当前键对应的哈希，再通过 bucketMask 和 add 函数拿到该键值对所在的桶序号和哈希最上面的 8 位数字。

```go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
  if h.flags&hashWriting != 0 {  // 检测是否并发写，map不是gorountine安全的
		throw("concurrent map read and map write")
	}
  hash := t.hasher(key, uintptr(h.hash0)) 
	m := bucketMask(h.B)
  b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
  // 如果老的bucket没有被移动完，那么去老的bucket中寻找
  if c := h.oldbuckets; c != nil {
		if !h.sameSizeGrow() {
			// There used to be half as many buckets; mask down one more power of two.
			m >>= 1
		}
		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
		if !evacuated(oldb) {
			b = oldb
		}
  }
  // 寻找过程：不断比对tophash和key
	top := tophash(hash)
bucketloop:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if alg.equal(key, k) {
				v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
				return v
			}
		}
	}
	return unsafe.Pointer(&zeroVal[0])
}
```

在 bucketloop 循环中，哈希会依次遍历正常桶和溢出桶中的数据，它会比较这 8 位数字和桶中存储的 tophash，每一个桶都存储键对应的 tophash，每一次读写操作都会与桶中所有的 tophash 进行比较，用于选择桶序号的是哈希的最低几位，而用于加速访问的是哈希的高 8 位，这种设计能够减少同一个桶中有大量相等 tophash 的概率。

![访问map](/Golang-map探险/访问map.png)

2. 使用for range遍历map

在遍历哈希表时，编译器会使用`runtime.mapiterinit`和`runtime.mapiternext`两个运行时函数重写原始的 for range 循环：

```go
ha := a
hit := hiter(n.Type)
th := hit.Type
mapiterinit(typename(t), ha, &hit)
for ; hit.key != nil; mapiternext(&hit) {
    key := *hit.key
    val := *hit.val
}
```

上述代码是 `for key, val := range hash {}` 生成的，在 `cmd/compile/internal/gc.walkrange` 函数处理 `TMAP` 节点时会根据接受 range 返回值的数量在循环体中插入需要的赋值语句：

![访问map](/Golang-map探险/range遍历map.png)

这三种不同的情况会分别向循环体插入不同的赋值语句。遍历哈希表时会使用 runtime.mapiterinit 函数初始化遍历开始的元素：

```go
func mapiterinit(t *maptype, h *hmap, it *hiter) {
	it.t = t
	it.h = h
	it.B = h.B
	it.buckets = h.buckets

	r := uintptr(fastrand())
	it.startBucket = r & bucketMask(h.B)
	it.offset = uint8(r >> h.B & (bucketCnt - 1))
	it.bucket = it.startBucket
	mapiternext(it)
}
```

该函数会初始化 hiter 结构体中的字段，并通过 runtime.fastrand 生成一个随机数帮助我们随机选择一个桶开始遍历。Go 团队在设计哈希表的遍历时就不想让使用者依赖固定的遍历顺序，所以引入了随机数保证遍历的随机性。

遍历哈希会使用 runtime.mapiternext 函数，我们在这里简化了很多逻辑，省去了一些边界条件以及哈希表扩容时的兼容操作，这里只需要关注处理遍历逻辑的核心代码，我们会将该函数分成桶的选择和桶内元素的遍历两部分进行分析，首先是桶的选择过程：

```go
func mapiternext(it *hiter) {
	h := it.h
	t := it.t
	bucket := it.bucket
	b := it.bptr
	i := it.i
	alg := t.key.alg

next:
	if b == nil {
		if bucket == it.startBucket && it.wrapped {
			it.key = nil
			it.value = nil
			return
		}
		b = (*bmap)(add(it.buckets, bucket*uintptr(t.bucketsize)))
		bucket++
		if bucket == bucketShift(it.B) {
			bucket = 0
			it.wrapped = true
		}
		i = 0
	}
```

这段代码主要有两个作用：

1. 在待遍历的桶为空时选择需要遍历的新桶；
2. 在不存在待遍历的桶时返回 (nil, nil) 键值对并中止遍历过程；

runtime.mapiternext 函数中第二段代码的主要作用就是从桶中找到下一个遍历的元素，在大多数情况下都会直接操作内存获取目标键值的内存地址，不过如果哈希表处于扩容期间就会调用 runtime.mapaccessK 函数获取键值对：

```go
	for ; i < bucketCnt; i++ {
		offi := (i + it.offset) & (bucketCnt - 1)
		k := add(unsafe.Pointer(b), dataOffset+uintptr(offi)*uintptr(t.keysize))
		v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+uintptr(offi)*uintptr(t.valuesize))
		if (b.tophash[offi] != evacuatedX && b.tophash[offi] != evacuatedY) ||
			!(t.reflexivekey() || alg.equal(k, k)) {
			it.key = k
			it.value = v
		} else {
			rk, rv := mapaccessK(t, h, k)
			it.key = rk
			it.value = rv
		}
		it.bucket = bucket
		it.i = i + 1
		return
	}
	b = b.overflow(t)
	i = 0
	goto next
}
```

当上述函数已经遍历了正常桶，就会通过`runtime.bmap.overflow`获取溢出桶依次进行遍历。

简单总结一下哈希表遍历的顺序，首先会选出一个正常桶开始遍历，随后遍历对应的所有溢出桶，最后依次按照索引顺序遍历哈希表中其他的桶，直到所有的桶都被遍历完成。

![map遍历](/Golang-map探险/map遍历.png)

### 写入map

在`cmd/compile/internal/gc/walk.go#walkexpr`方法的`OINDEXMAP`分支中

```go
if n.IndexMapLValue() {
  // This m[k] expression is on the left-hand side of an assignment.
  fast := mapfast(t)
  if fast == mapslow {
    // standard version takes key by reference.
    // order.expr made sure key is addressable.
    key = nod(OADDR, key, nil)
  }
  n = mkcall1(mapfn(mapassign[fast], t), nil, init, typename(t), map_, key)
} 
```

当m[k]实在赋值的左边时，会转换成`runtime.mapassign`函数的调用；首先是函数会根据传入的键拿到对应的哈希和桶

```go
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
  if h.flags&hashWriting != 0 {
    throw("concurrent map writes")
  }
  hash := t.hasher(key, uintptr(h.hash0))

  // Set hashWriting after calling t.hasher, since t.hasher may panic,
  // in which case we have not actually done a write.
  h.flags ^= hashWriting

again:
  bucket := hash & bucketMask(h.B)
  b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + bucket*uintptr(t.bucketsize)))
  top := tophash(hash)
```

然后通过遍历比较桶中存储的 tophash 和键的哈希，如果找到了相同结果就会返回目标位置的地址。其中 inserti 表示目标元素的在桶中的索引，insertk 和 elem 分别表示键值对的地址，获得目标地址之后会通过算术计算寻址获得键值对 k 和 elem:

```go
	var inserti *uint8
	var insertk unsafe.Pointer
	var elem unsafe.Pointer
bucketloop:
  for {
    for i := uintptr(0); i < bucketCnt; i++ {
      // 遍历 8 个 bucket 中的元素
      if b.tophash[i] != top {
        if isEmpty(b.tophash[i]) && inserti == nil {
          // 如果这个槽位没有被占，说明可以往这里塞 key 和 value
          inserti = &b.tophash[i]  // tophash 的插入位置
          insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
          elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
        }
        if b.tophash[i] == emptyRest {
          break bucketloop
        }
        continue
      }
      k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
      if t.indirectkey() {
        k = *((*unsafe.Pointer)(k))
      }
      // 如果相同的 hash 位置的 key 和要插入的 key 字面上不相等
      // 如果两个 key 的首八位后最后八位哈希值一样，就会进行其值比较
      // 算是一种哈希碰撞吧 
      if !t.key.equal(key, k) {
        continue
      }
      // already have a mapping for key. Update it.
      // 对应的位置已经有 key 了，直接更新就行
      if t.needkeyupdate() {
        typedmemmove(t.key, k, key)
      }
      elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
      goto done
    }
    // bucket 的 8 个槽没有满足条件的能插入或者能更新的，去 overflow 里继续找
    ovf := b.overflow(t)
    // 如果 overflow 为 nil，说明到了 overflow 链表的末端了
    if ovf == nil {
      break
    }
    b = ovf
  }
```

上述的 for 循环会依次遍历正常桶和溢出桶中存储的数据，整个过程会分别判断 tophash 是否相等、key 是否相等，遍历结束后会从循环中跳出。

如果当前桶已经满了，哈希会调用`runtime.hmap.newoverflow`创建新桶或者使用`runtime.hmap`预先在noverflow中创建好的桶来保存数据，新创建的桶不仅会被追加到已有桶的末尾，还会增加哈希表的 noverflow 计数器。

```go
  if inserti == nil {
    // The current bucket and all the overflow buckets connected to it are full, allocate a new one.
    // 前面在桶里找的时候，没有找到能塞这个 tophash 的位置
    // 说明当前所有 buckets 都是满的，分配一个新的 bucket
    newb := h.newoverflow(t, b)
    inserti = &newb.tophash[0]
    insertk = add(unsafe.Pointer(newb), dataOffset)
    elem = add(insertk, bucketCnt*uintptr(t.keysize))
  }
  // store new key/elem at insert position
  if t.indirectkey() {
    kmem := newobject(t.key)
    *(*unsafe.Pointer)(insertk) = kmem
    insertk = kmem
  }
  if t.indirectelem() {
    vmem := newobject(t.elem)
    *(*unsafe.Pointer)(elem) = vmem
  }

  typedmemmove(t.key, insertk, key)
  *inserti = top
  h.count++
```

如果当前键值对在哈希中不存在，哈希会为新键值对规划存储的内存地址，通过`runtime.typedmemmove`将键移动到对应的内存空间中并返回键对应值的地址 val。如果当前键值对在哈希中存在，那么就会直接返回目标区域的内存地址，哈希并不会在`runtime.mapassign`这个运行时函数中将值拷贝到桶中，该函数只会返回内存地址，真正的赋值操作是在编译期间插入的：

```
00018 (+5) CALL runtime.mapassign_fast64(SB)
00020 (5) MOVQ 24(SP), DI               ;; DI = &value
00026 (5) LEAQ go.string."88"(SB), AX   ;; AX = &"88"
00027 (5) MOVQ AX, (DI)                 ;; *DI = AX
```

赋值的最后一步实际上是编译器额外生成的汇编指令来完成的，可见靠 runtime 有些工作是没有做完的。这里和 go 在函数调用时插入 prologue 和 epilogue 是类似的。编译器和 runtime 配合，才能完成一些复杂的工作。

### map扩容

在`mapassign`中还有一段关于扩容的逻辑

```go
// If we hit the max load factor or we have too many overflow buckets,
// and we're not already in the middle of growing, start growing.
if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
  hashGrow(t, h)
  goto again // Growing the table invalidates everything, so try again
}
```

```go
// overLoadFactor reports whether count items placed in 1<<B buckets is over loadFactor.
func overLoadFactor(count int, B uint8) bool {
  return count > bucketCnt && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)
}
```

```go
// tooManyOverflowBuckets reports whether noverflow buckets is too many for a map with 1<<B buckets.
// Note that most of these overflow buckets must be in sparse use;
// if use was dense, then we'd have already triggered regular map growth.
func tooManyOverflowBuckets(noverflow uint16, B uint8) bool {
  // If the threshold is too low, we do extraneous work.
  // If the threshold is too high, maps that grow and shrink can hold on to lots of unused memory.
  // "too many" means (approximately) as many overflow buckets as regular buckets.
  // See incrnoverflow for more details.
  if B > 15 {
    B = 15
  }
  // The compiler doesn't see here that B < 16; mask B to generate shorter shift code.
  return noverflow >= uint16(1)<<(B&15)
}
```

可以看到有2种条件会触发扩容

1. 是不是已经到了 load factor 的临界点，即元素个数 >= 桶个数 * 6.5，这时候说明大部分的桶可能都快满了，如果插入新元素，有大概率需要挂在 overflow 的桶上
2. overflow 的桶是不是太多了，当 bucket 总数 < 2 ^ 15 时，如果 overflow 的 bucket 总数 >= bucket 的总数，那么我们认为 overflow 的桶太多了。当 bucket 总数 >= 2 ^ 15 时，那我们直接和 2 ^ 15 比较，overflow 的 bucket >= 2 ^ 15 时，即认为溢出桶太多了。为啥会导致这种情况呢？是因为我们对 map 一边插入，一边删除，会导致其中很多桶出现空洞，这样使得 bucket 使用率不高，值存储得比较稀疏。在查找时效率会下降

对于第2点我在看其他大佬的博客时有点迷惑，这里说一下我自己的理解，可能不太正确

什么情况下会出现 overflow 的 bucket 总数 >= bucket 的总数？

看了前面`mapassign`的过程，在map中插入值的时候会先在正常桶中找到空位置，而且尽量找靠前的空位置，只有正常桶的8个位置都满了之后才会创建新的overflow桶，而且有条件1的存在，也就是元素个数 >= 桶个数 * 6.5会触发扩容，在以上条件下，为什么还会出现overflow 的 bucket 总数 >= bucket 的总数？

在Draveness大佬的博客中[关于扩容这段](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap/#%E6%89%A9%E5%AE%B9)中提到了一个[issue](https://github.com/golang/go/issues/16070)，在这个issue中给出了一段使用map的代码，大概就是在map中随机添加一些值，然后又随机删除(源代码中并不是随机删除的，但是并不是添加完后立即删除，所以我就理解成随机删除了)，最后发现map占用的空间会越来越大(也就是内存泄漏)，其实这个问题就回答了前面的疑问

再分析一下原因：不停地插入、删除元素。先插入很多元素，导致创建了很多 bucket，但是装载因子达不到第 1 点的临界值，未触发扩容来缓解这种情况。之后，删除元素降低元素总数量，再插入很多元素，导致创建很多的 overflow bucket，但就是不会触犯第 1 点的规定，你能拿我怎么办？overflow bucket 数量太多，导致 key 会很分散，查找插入效率低得吓人，因此出台第 2 点规定。这就像是一座空城，房子很多，但是住户很少，都分散了，找起人来很困难。

为什么key很分散，查找插入效率低得吓人？

这个问题其实很简单，可能我当时脑子转不过弯了。查找和插入都会定位到一个正常桶，如果出现前面说的，不停插入、删除导致key很分散，也就是这个正常桶后面链着很多的overflow桶，这时候你查找和插入找位置的时候就需要遍历这条bucket链，所以效率会降低。

所以对于命中条件 1，2 的限制，都会发生扩容。但是扩容的策略并不相同：

* 对于条件 1，元素太多，而 bucket 数量太少，很简单：将 B 加 1，bucket 最大数量（2^B）直接变成原来 bucket 数量的 2 倍。于是，就有新老 bucket 了。注意，这时候元素都在老 bucket 里，还没迁移到新的 bucket 来。而且，新 bucket 只是最大数量变为原来最大数量（2^B）的 2 倍（2^B * 2）
* 对于条件 2，其实元素没那么多，但是 overflow bucket 数特别多，说明很多 bucket 都没装满。解决办法就是开辟一个新 bucket 空间，将老 bucket 中的元素移动到新 bucket，使得同一个 bucket 中的 key 排列地更紧密。这样，原来，在 overflow bucket 中的 key 可以移动到 bucket 中来。结果是节省空间，提高 bucket 利用率，map 的查找和插入效率自然就会提升


扩容的入口是`runtime.hashGrow`：

```go
func hashGrow(t *maptype, h *hmap) {
  // B+1 相当于是原来 2 倍的空间
  bigger := uint8(1)
  // 对应条件 2
	if !overLoadFactor(h.count+1, h.B) {
    // 进行等量的内存扩容，所以 B 不变
		bigger = 0
		h.flags |= sameSizeGrow
  }
  // 将老 buckets 挂到 buckets 上
  oldbuckets := h.buckets
  // 申请新的 buckets 空间
	newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)

  // 提交 grow 的动作
	h.B += bigger
	h.flags = flags
	h.oldbuckets = oldbuckets
  h.buckets = newbuckets
  // 搬迁进度为 0
  h.nevacuate = 0
  // overflow buckets 数为 0
	h.noverflow = 0

	h.extra.oldoverflow = h.extra.overflow
	h.extra.overflow = nil
	h.extra.nextOverflow = nextOverflow
}
```

哈希在扩容的过程中会通过 runtime.makeBucketArray 创建一组新桶和预创建的溢出桶，随后将原有的桶数组设置到 oldbuckets 上并将新的空桶设置到 buckets 上，溢出桶也使用了相同的逻辑更新

我们在`runtime.hashGrow`中还看不出来等量扩容和翻倍扩容的太多区别，等量扩容创建的新桶数量只是和旧桶一样，该函数中只是创建了新的桶，并没有对数据进行拷贝和转移

再来看看真正执行搬迁工作的`growWork()`函数

```go
func growWork(t *maptype, h *hmap, bucket uintptr) {
	// make sure we evacuate the oldbucket corresponding
  // to the bucket we're about to use
  // 确认搬迁老的 bucket 对应正在使用的 bucket
	evacuate(t, h, bucket&h.oldbucketmask())

  // evacuate one more oldbucket to make progress on growing
  // 再搬迁一个 bucket，以加快搬迁进程
	if h.growing() {
		evacuate(t, h, h.nevacuate)
	}
}
```

接下来，我们集中所有的精力在搬迁的关键函数 evacuate

```go
func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
  // 定位老的 bucket 地址
  b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
  // 结果是 2^B，如 B = 5，结果为32
  newbit := h.noldbuckets()
  // 如果 b 没有被搬迁过
  if !evacuated(b) {
    // TODO: reuse overflow buckets instead of using new ones, if there
    // is no iterator using the old buckets.  (If !oldIterator.)

    // xy contains the x and y (low and high) evacuation destinations.
    // xy 包含的是移动的目标
    // x 表示新 bucket 数组的前(low)半部分
    // y 表示新 bucket 数组的后(high)半部分
    var xy [2]evacDst
    x := &xy[0]
    x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
    x.k = add(unsafe.Pointer(x.b), dataOffset)
    x.e = add(x.k, bucketCnt*uintptr(t.keysize))

    // 如果不是等 size 扩容，前后 bucket 序号有变
    // 使用 y 来进行搬迁
    if !h.sameSizeGrow() {
      // Only calculate y pointers if we're growing bigger.
      // Otherwise GC can see bad pointers.
      // 如果 map 大小(hmap.B)增大了，那么我们只计算 y
      // 否则 GC 可能会看到损坏的指针
      y := &xy[1]
      y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize)))
      y.k = add(unsafe.Pointer(y.b), dataOffset)
      y.e = add(y.k, bucketCnt*uintptr(t.keysize))
    }

    // 遍历所有的 bucket，包括 overflow buckets
    // b 是老的 bucket 地址
    for ; b != nil; b = b.overflow(t) {
      k := add(unsafe.Pointer(b), dataOffset)
      e := add(k, bucketCnt*uintptr(t.keysize))
      // 遍历 bucket 中的所有 cell
      for i := 0; i < bucketCnt; i, k, e = i+1, add(k, uintptr(t.keysize)), add(e, uintptr(t.elemsize)) {
        // 当前 cell 的 top hash 值
        top := b.tophash[i]
        // 如果 cell 为空，即没有 key
        if isEmpty(top) {
          // 那就标志它被"搬迁"过
          b.tophash[i] = evacuatedEmpty
          continue
        }
        // 正常不会出现这种情况
        // 未被搬迁的 cell 只可能是 empty 或是
        // 正常的 top hash（大于 minTopHash）
        if top < minTopHash {
          throw("bad map state")
        }
        k2 := k
        // 如果 key 是指针，则解引用
        if t.indirectkey() {
          k2 = *((*unsafe.Pointer)(k2))
        }
        var useY uint8
        // 如果不是等量扩容
        if !h.sameSizeGrow() {
          // Compute hash to make our evacuation decision (whether we need
          // to send this key/elem to bucket x or bucket y).
          // 计算哈希，以判断我们的数据要转移到哪一部分的 bucket
          // 可能是 x 部分，也可能是 y 部分
          hash := t.hasher(k2, uintptr(h.hash0))
          if h.flags&iterator != 0 && !t.reflexivekey() && !t.key.equal(k2, k2) {
            // If key != key (NaNs), then the hash could be (and probably
            // will be) entirely different from the old hash. Moreover,
            // it isn't reproducible. Reproducibility is required in the
            // presence of iterators, as our evacuation decision must
            // match whatever decision the iterator made.
            // Fortunately, we have the freedom to send these keys either
            // way. Also, tophash is meaningless for these kinds of keys.
            // We let the low bit of tophash drive the evacuation decision.
            // We recompute a new random tophash for the next level so
            // these keys will get evenly distributed across all buckets
            // after multiple grows.
            // 为什么要加 reflexivekey 的判断，可以参考这里:
            // https://go-review.googlesource.com/c/go/+/1480
            // key != key，只有在 float 数的 NaN 时会出现
            // 比如:
            // n1 := math.NaN()
            // n2 := math.NaN()
            // fmt.Println(n1, n2)
            // fmt.Println(n1 == n2)
            // 这种情况下 n1 和 n2 的哈希值也完全不一样
            // 这里官方表示这种情况是不可复现的
            // 需要在 iterators 参与的情况下才能复现
            // 但是对于这种 key 我们也可以随意对其目标进行发配
            // 同时 tophash 对于 NaN 也没啥意义
            // 还是按正常的情况下算一个随机的 tophash
            // 然后公平地把这些 key 平均分布到各 bucket 就好
            useY = top & 1 // 让这个 key 50% 概率去 Y 半区
            top = tophash(hash)
          } else {
            // 这里写的比较 trick
            // 比如当前有 8 个桶
            // 那么如果 hash & 8 != 0
            // 那么说明这个元素的 hash 这种形式
            // xxx1xxx
            // 而扩容后的 bucketMask 是
            //    1111
            // 所以实际上这个就是
            // xxx1xxx & 1000 > 0
            // 说明这个元素在扩容后一定会去下半区，即Y部分
            // 所以就是 useY 了
            if hash&newbit != 0 {
              useY = 1
            }
          }
        }

        if evacuatedX+1 != evacuatedY || evacuatedX^1 != evacuatedY {
          throw("bad evacuatedN")
        }

        b.tophash[i] = evacuatedX + useY // evacuatedX + 1 == evacuatedY
        dst := &xy[useY]                 // evacuation destination 移动目标

        if dst.i == bucketCnt {
          dst.b = h.newoverflow(t, dst.b)
          dst.i = 0
          dst.k = add(unsafe.Pointer(dst.b), dataOffset)
          dst.e = add(dst.k, bucketCnt*uintptr(t.keysize))
        }
        dst.b.tophash[dst.i&(bucketCnt-1)] = top // mask dst.i as an optimization, to avoid a bounds check
        if t.indirectkey() {
          *(*unsafe.Pointer)(dst.k) = k2 // 拷贝指针
        } else {
          typedmemmove(t.key, dst.k, k) // 拷贝值
        }
        if t.indirectelem() {
          *(*unsafe.Pointer)(dst.e) = *(*unsafe.Pointer)(e)
        } else {
          typedmemmove(t.elem, dst.e, e)
        }
        dst.i++
        // These updates might push these pointers past the end of the
        // key or elem arrays.  That's ok, as we have the overflow pointer
        // at the end of the bucket to protect against pointing past the
        // end of the bucket.
        dst.k = add(dst.k, uintptr(t.keysize))
        dst.e = add(dst.e, uintptr(t.elemsize))
      }
    }
    // Unlink the overflow buckets & clear key/elem to help GC.
    if h.flags&oldIterator == 0 && t.bucket.ptrdata != 0 {
      b := add(h.oldbuckets, oldbucket*uintptr(t.bucketsize))
      // Preserve b.tophash because the evacuation
      // state is maintained there.
      ptr := add(b, dataOffset)
      n := uintptr(t.bucketsize) - dataOffset
      memclrHasPointers(ptr, n)
    }
  }

  if oldbucket == h.nevacuate {
    advanceEvacuationMark(h, t, newbit)
  }
}
```

`runtime.evacuate` 最后会调用`runtime.advanceEvacuationMark` 增加哈希的`nevacuate`计数器并在所有的旧桶都被分流后清空哈希的 oldbuckets 和 oldoverflow：

```go
func advanceEvacuationMark(h *hmap, t *maptype, newbit uintptr) {
  h.nevacuate++
  // Experiments suggest that 1024 is overkill by at least an order of magnitude.
  // Put it in there as a safeguard anyway, to ensure O(1) behavior.
  stop := h.nevacuate + 1024
  if stop > newbit {
    stop = newbit
  }
  for h.nevacuate != stop && bucketEvacuated(t, h, h.nevacuate) {
    h.nevacuate++
  }
  if h.nevacuate == newbit { // newbit == # of oldbuckets
    // Growing is all done. Free old main bucket array.
    // 大小增长全部结束。释放老的 bucket array
    h.oldbuckets = nil
    // Can discard old overflow buckets as well.
    // If they are still referenced by an iterator,
    // then the iterator holds a pointers to the slice.
    // 同样可以丢弃老的 overflow buckets
    // 如果它们还被迭代器所引用的话
    // 迭代器会持有一份指向 slice 的指针
    if h.extra != nil {
      h.extra.oldoverflow = nil
    }
    h.flags &^= sameSizeGrow
  }
}
```

扩容并不是一次就完成的，而是渐进的

在`runtime.mapassign`函数中也省略了一段逻辑，当哈希表正在处于扩容状态时，每次向哈希表写入值时都会触发`runtime.growWork`增量拷贝哈希表中的内容：

```go
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
...
again:
	bucket := hash & bucketMask(h.B)
	if h.growing() {
		growWork(t, h, bucket)
	}
...
}
```
当然在`runtime.mapdelete`函数中也会触发`runtime.growWork`增量拷贝哈希表中的内容：
```go
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {
...
	bucket := hash & bucketMask(h.B)
	if h.growing() {
		growWork(t, h, bucket)
	}
...
}
```

之前在分析哈希表访问函数`runtime.mapaccess1`时其实省略了扩容期间获取键值对的逻辑，当哈希表的 oldbuckets 存在时，会先定位到旧桶并在该桶没有被分流时从中获取键值对。

```go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
  ...
  alg := t.key.alg
  hash := alg.hash(key, uintptr(h.hash0))
  m := bucketMask(h.B)
  b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
  if c := h.oldbuckets; c != nil {
    if !h.sameSizeGrow() {
      m >>= 1
    }
    oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
    if !evacuated(oldb) {
      b = oldb
    }
  }
  bucketloop:
  ...
}
```

因为旧桶中的元素还没有被 runtime.evacuate 函数分流，其中还保存着我们需要使用的数据，所以旧桶会替代新创建的空桶提供数据。

简单总结一下哈希表扩容的设计和原理，哈希在存储元素过多时会触发扩容操作，每次都会将桶的数量翻倍，扩容过程不是原子的，而是通过 runtime.growWork 增量触发的，在扩容期间访问哈希表时会使用旧桶，向哈希表写入数据时会触发旧桶元素的分流。除了这种正常的扩容之外，为了解决大量写入、删除造成的内存泄漏问题，哈希引入了 sameSizeGrow 这一机制，在出现较多溢出桶时会整理哈希的内存减少空间的占用。

sameSizeGrow的图示

![map遍历](/Golang-map探险/SameSizeGrow.png)

biggerSizeGrow的图示

![map遍历](/Golang-map探险/BiggerSizeGrow.png)

### map删除

如果想要删除哈希中的元素，就需要使用 Go 语言中的 delete 关键字，这个关键字的唯一作用就是将某一个键对应的元素从哈希表中删除，无论是该键对应的值是否存在，这个内建的函数都不会返回任何的结果。

在编译期间，`delete`关键字会被转换成操作为`ODELETE`的节点，而`cmd/compile/internal/gc.walkexpr`会将`ODELETE`节点转换成`runtime.mapdelete`函数簇中的一个，包括 `runtime.mapdelete`、`mapdelete_faststr`、`mapdelete_fast32` 和 `mapdelete_fast64`：

```go
func walkexpr(n *Node, init *Nodes) *Node {
	switch n.Op {
	case ODELETE:
		init.AppendNodes(&n.Ninit)
		map_ := n.List.First()
		key := n.List.Second()
		map_ = walkexpr(map_, init)
		key = walkexpr(key, init)

		t := map_.Type
		fast := mapfast(t)
		if fast == mapslow {
			key = nod(OADDR, key, nil)
		}
		n = mkcall1(mapfndel(mapdelete[fast], t), nil, init, typename(t), map_, key)
	}
}
```

这些函数的实现其实差不多，我们挑选其中的`runtime.mapdelete`分析一下。哈希表的删除逻辑与写入逻辑很相似，只是触发哈希的删除需要使用关键字，如果在删除期间遇到了哈希表的扩容，就会分流桶中的元素，分流结束之后会找到桶中的目标元素完成键值对的删除工作。

```go
  func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {
  if h == nil || h.count == 0 {
    if t.hashMightPanic() {
      t.hasher(key, 0) // see issue 23734
    }
    return
  }
  if h.flags&hashWriting != 0 {
    throw("concurrent map writes")
  }

  hash := t.hasher(key, uintptr(h.hash0))

  // Set hashWriting after calling t.hasher, since t.hasher may panic,
  // in which case we have not actually done a write (delete).
  h.flags ^= hashWriting

  // 按低 8 位 hash 值选择 bucket
  bucket := hash & bucketMask(h.B)
  if h.growing() {
    growWork(t, h, bucket)
  }
  // 按上面算出的桶的索引，找到 bucket 的内存地址
  // 并强制转换为需要的 bmap 结构
  b := (*bmap)(add(h.buckets, bucket*uintptr(t.bucketsize)))
  bOrig := b
  // 高 8 位 hash 值
  top := tophash(hash)
  search:
  for ; b != nil; b = b.overflow(t) {
    for i := uintptr(0); i < bucketCnt; i++ {
      // 8 个槽位，分别对比 tophash
      // 没找到的话就去外围 for 循环的 overflow 链表中继续查找
      if b.tophash[i] != top {
        if b.tophash[i] == emptyRest {
          break search
        }
        continue
      }
        // 计算 k 所在的槽位的内存地址
      k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
      k2 := k
      // 如果 key > 128 字节
      if t.indirectkey() {
        k2 = *((*unsafe.Pointer)(k2))
      }
      if !t.key.equal(key, k2) {
        continue
      }
      // Only clear key if there are pointers in it.
      // 如果 key 中是指针，那么清空 key 的内容
      if t.indirectkey() {
        *(*unsafe.Pointer)(k) = nil
      } else if t.key.ptrdata != 0 {
        memclrHasPointers(k, t.key.size)
      }
      // 计算 value 所在的内存地址
      e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
      if t.indirectelem() {
        *(*unsafe.Pointer)(e) = nil
      } else if t.elem.ptrdata != 0 {
        memclrHasPointers(e, t.elem.size)
      } else {
        memclrNoHeapPointers(e, t.elem.size)
      }
      // 设置 tophash[i] = 0
      b.tophash[i] = emptyOne
      // If the bucket now ends in a bunch of emptyOne states,
      // change those to emptyRest states.
      // It would be nice to make this a separate function, but
      // for loops are not currently inlineable.
      if i == bucketCnt-1 {
        if b.overflow(t) != nil && b.overflow(t).tophash[0] != emptyRest {
          goto notLast
        }
      } else {
        if b.tophash[i+1] != emptyRest {
          goto notLast
        }
      }
      for {
        b.tophash[i] = emptyRest
        if i == 0 {
          if b == bOrig {
            break // beginning of initial bucket, we're done.
          }
          // Find previous bucket, continue at its last entry.
          c := b
          for b = bOrig; b.overflow(t) != c; b = b.overflow(t) {
          }
          i = bucketCnt - 1
        } else {
          i--
        }
        if b.tophash[i] != emptyOne {
          break
        }
      }
    notLast:
      h.count--
      // Reset the hash seed to make it more difficult for attackers to
      // repeatedly trigger hash collisions. See issue 25237.
      if h.count == 0 {
        h.hash0 = fastrand()
      }
      break search
    }
  }

  if h.flags&hashWriting == 0 {
    throw("concurrent map writes")
  }
  h.flags &^= hashWriting
  }
```

我们其实只需要知道`delete`关键字在编译期间经过类型检查和中间代码生成阶段被转换成`runtime.mapdelete`函数簇中的一员，用于处理删除逻辑的函数与哈希表的`runtime.mapassign`几乎完全相同，不太需要刻意关注。

### indirectkey 和 indirectvalue

在上面的代码中我们见过无数次的 indirectkey 和 indirectvalue。indirectkey 和 indirectvalue 在 map 里实际存储的是指针，会造成 GC 扫描时，扫描更多的对象。至于是否是 indirect，依然是由编译器来决定的，依据是:

* key > 128 字节时，indirectkey = true
* value > 128 字节时，indirectvalue = true

### overflow 处理

思路: 从 nextOverflow 中拿 overflow bucket，如果拿到，就放进 hmap.extra.overflow 数组，并让 bmap 的 overflow pointer 指向这个 bucket。

如果没找到，那就 new 一个。

```go
func (h *hmap) newoverflow(t *maptype, b *bmap) *bmap {
	var ovf *bmap
	if h.extra != nil && h.extra.nextOverflow != nil {
		// We have preallocated overflow buckets available.
		// See makeBucketArray for more details.
		ovf = h.extra.nextOverflow
		if ovf.overflow(t) == nil {
			// We're not at the end of the preallocated overflow buckets. Bump the pointer.
			h.extra.nextOverflow = (*bmap)(add(unsafe.Pointer(ovf), uintptr(t.bucketsize)))
		} else {
			// This is the last preallocated overflow bucket.
			// Reset the overflow pointer on this bucket,
			// which was set to a non-nil sentinel value.
			ovf.setoverflow(t, nil)
			h.extra.nextOverflow = nil
		}
	} else {
		ovf = (*bmap)(newobject(t.bucket))
	}
	h.incrnoverflow()
	if t.bucket.ptrdata == 0 {
		h.createOverflow()
		*h.extra.overflow = append(*h.extra.overflow, ovf)
	}
	b.setoverflow(t, ovf)
	return ovf
}

func (h *hmap) createOverflow() {
    if h.extra == nil {
        h.extra = new(mapextra)
    }
    if h.extra.overflow == nil {
        h.extra.overflow = new([]*bmap)
    }
}

// incrnoverflow increments h.noverflow.
// noverflow counts the number of overflow buckets.
// This is used to trigger same-size map growth.
// See also tooManyOverflowBuckets.
// To keep hmap small, noverflow is a uint16.
// When there are few buckets, noverflow is an exact count.
// When there are many buckets, noverflow is an approximate count.
func (h *hmap) incrnoverflow() {
	// We trigger same-size map growth if there are
	// as many overflow buckets as buckets.
	// We need to be able to count to 1<<h.B.
	if h.B < 16 {
		h.noverflow++
		return
	}
	// Increment with probability 1/(1<<(h.B-15)).
	// When we reach 1<<15 - 1, we will have approximately
	// as many overflow buckets as buckets.
	mask := uint32(1)<<(h.B-15) - 1
	// Example: if h.B == 18, then mask == 7,
	// and fastrand & 7 == 0 with probability 1/8.
	if fastrand()&mask == 0 {
		h.noverflow++
	}
}
```

小结

Go 语言使用拉链法来解决哈希碰撞的问题实现了哈希表，它的访问、写入和删除等操作都在编译期间转换成了运行时的函数或者方法。哈希在每一个桶中存储键对应哈希的前 8 位，当对哈希进行操作时，这些 tophash 就成为可以帮助哈希快速遍历桶中元素的缓存。

哈希表的每个桶都只能存储 8 个键值对，一旦当前哈希的某个桶超出 8 个，新的键值对就会存储到哈希的溢出桶中。随着键值对数量的增加，溢出桶的数量和哈希的装载因子也会逐渐升高，超过一定范围就会触发扩容，扩容会将桶的数量翻倍，元素再分配的过程也是在调用写操作时增量进行的，不会造成性能的瞬时巨大抖动。

### 参考资料

* [深度解密Go语言之map](https://www.cnblogs.com/qcrao-2018/p/10903807.html)
* [哈希表](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap/)
* [map.md](https://github.com/cch123/golang-notes/blob/master/map.md)

