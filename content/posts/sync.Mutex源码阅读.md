---
title: "sync.Mutex源码阅读"
date: 2021-07-15T01:08:21+08:00
draft: false
original: true
categories: 
  - Golang
tags: 
  - Golang基础 
---

Mutex的定义

```go
type Mutex struct {
	state int32  // 当前互斥锁的状态
	sema  uint32 // 用于控制锁状态的信号量
}
```

其中state的低3位表示如下

```go
const (
	mutexLocked = 1 << iota // mutex is locked
	mutexWoken
	mutexStarving
)
```

![](/syncMutex源码阅读/01.png)

- `mutexLocked` — 表示互斥锁的锁定状态；
- `mutexWoken` — 表示从正常模式被从唤醒；
- `mutexStarving` — 当前的互斥锁进入饥饿状态；
- `waitersCount` — 当前互斥锁上等待的 Goroutine 个数；

<!--more-->
# 加锁过程

通过Mutex#Lock()方法进行加锁

```go
// Lock locks m.
// If the lock is already in use, the calling goroutine
// blocks until the mutex is available.
func (m *Mutex) Lock() {
	// Fast path: grab unlocked mutex.
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		if race.Enabled {
			race.Acquire(unsafe.Pointer(m))
		}
		return
	}
	// Slow path (outlined so that the fast path can be inlined)
	m.lockSlow()
}
```

如果Mutex没有被其他线程抢占，Lock()方法会先尝试使用atomic.CompareAndSwapInt32()方法将state字段的最低位置为mutexLocked，如果成功了说明加锁成功就可以之间返回。

如果Mutex存在其他线程抢占，就会调用lockSlow()方法。

```go
func (m *Mutex) lockSlow() {
	var waitStartTime int64
	starving := false
	awoke := false
	iter := 0
	old := m.state
	for {
		// Don't spin in starvation mode, ownership is handed off to waiters
		// so we won't be able to acquire the mutex anyway.
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
			// Active spinning makes sense.
			// Try to set mutexWoken flag to inform Unlock
			// to not wake other blocked goroutines.
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true
			}
			runtime_doSpin()
			iter++
			old = m.state
			continue
		}
		...
	}
```

old&(mutexLocked|mutexStarving) == mutexLocked  表示Mutex被其他goroutine抢占着，并且Mutex的state不是处于mutexStarving(也就是没有goroutine等待该Mutex的时间超过1ms)

runtime_canSpin(iter) 判断是否能够进入自旋

```go
active_spin     = 4

// Active spinning for sync.Mutex.
//go:linkname sync_runtime_canSpin sync.runtime_canSpin
//go:nosplit
func sync_runtime_canSpin(i int) bool {
	// sync.Mutex is cooperative, so we are conservative with spinning.
	// Spin only few times and only if running on a multicore machine and
	// GOMAXPROCS>1 and there is at least one other running P and local runq is empty.
	// As opposed to runtime mutex we don't do passive spinning here,
	// because there can be work on global runq or on other Ps.
	if i >= active_spin || ncpu <= 1 || gomaxprocs <= int32(sched.npidle+sched.nmspinning)+1 {
		return false
	}
	if p := getg().m.p.ptr(); !runqempty(p) {
		return false
	}
	return true
}
```

所以runtime_canSpin(iter) 返回true需要满足如下条件

1. 运行在多 CPU 的机器上；
2. 当前 Goroutine 为了获取该锁进入自旋的次数小于四次；
3. 当前机器上至少存在一个正在运行的处理器 P 并且处理的运行队列为空；

当满足自旋条件时，如果满足

- !awoke  当前goroutine的awoke标识为false
- old&mutexWoken == 0  Mutex不处于mutexWoken
- old>>mutexWaiterShift != 0 Mutex上没有等待的goroutine

就尝试 atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken)，将Mutex的state改为mutexWoken，并将当前goroutine的awoke标识置为true

```go
if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
	atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
	awoke = true
}
```

然后调用runtime_doSpin()进行自旋

```go
func sync_runtime_doSpin() {
	procyield(active_spin_cnt)
}

TEXT runtime·procyield(SB),NOSPLIT,$0-0
	MOVL	cycles+0(FP), AX
again:
	PAUSE
	SUBL	$1, AX
	JNZ	again
	RET
```

自旋就是 让CPU执行 30 次的 PAUSE 指令，该指令只会占用 CPU 并消耗 CPU 时间

处理了自旋相关的特殊逻辑之后，互斥锁会根据上下文计算当前互斥锁最新的状态

```go
new := old
// Don't try to acquire starving mutex, new arriving goroutines must queue.
if old&mutexStarving == 0 {  // 如果Mutex的state不是处于mutexStarving，也就是没有goroutine等待该Mutex的时间超过1ms，当前goroutine就可以尝试去抢锁
	new |= mutexLocked
}
if old&(mutexLocked|mutexStarving) != 0 { //  如果Mutex被其他goroutine抢占着或者处于mutexStarving，也就是存在goroutine等待该Mutex的时间超过1ms，那么mutexWaiter+1
	new += 1 << mutexWaiterShift
}
// The current goroutine switches mutex to starvation mode.
// But if the mutex is currently unlocked, don't do the switch.
// Unlock expects that starving mutex has waiters, which will not
// be true in this case.
if starving && old&mutexLocked != 0 { // 如果当前 goroutine 的 starving 标识为true(在后面会看到什么时候会将starving设置成 true)，并且Mutex还是被其他goroutine抢占着，设置成mutexStarving
	new |= mutexStarving
}
// awoke为true则表明当前线程在上面自旋的时候，修改mutexWoken状态成功
if awoke {
	// The goroutine has been woken from sleep,
	// so we need to reset the flag in either case.
	if new&mutexWoken == 0 {
		throw("sync: inconsistent mutex state")
	}
	new &^= mutexWoken // 清除唤醒标志位
}
```

接下来会调用atomic.CompareAndSwapInt32(&m.state, old, new)，尝试修改Mutex的state的new

```go
if atomic.CompareAndSwapInt32(&m.state, old, new) {
	...
} else  {
	old = m.state
}
```

如果atomic.CompareAndSwapInt32()修改失败，那么一切回到原点，再继续走前面的了逻辑

如果atomic.CompareAndSwapInt32()修改成功，也就是意味着在 互斥锁根据上下文计算当前互斥锁最新的状态 的这段逻辑处理过程中，mutex的state没有被其他 goroutine 修改，也就是说下面这段逻辑执行开始到执行最后，old和Mutex的state是一致的

```go
new := old
// Don't try to acquire starving mutex, new arriving goroutines must queue.
if old&mutexStarving == 0 {  // 如果Mutex的state不是处于mutexStarving，也就是没有goroutine等待该Mutex的时间超过1ms，当前goroutine就可以尝试去抢锁
	new |= mutexLocked
}
if old&(mutexLocked|mutexStarving) != 0 { //  如果Mutex被其他goroutine抢占着或者处于mutexStarving，也就是存在goroutine等待该Mutex的时间超过1ms，那么mutexWaiter+1
	new += 1 << mutexWaiterShift
}
// The current goroutine switches mutex to starvation mode.
// But if the mutex is currently unlocked, don't do the switch.
// Unlock expects that starving mutex has waiters, which will not
// be true in this case.
if starving && old&mutexLocked != 0 { // 如果当前 goroutine 的 starving 标识为true(在后面会看到什么时候会将starving设置成 true)，并且Mutex还是被其他goroutine抢占着，设置成mutexStarving
	new |= mutexStarving
}
// awoke为true则表明当前线程在上面自旋的时候，修改mutexWoken状态成功
if awoke {
	// The goroutine has been woken from sleep,
	// so we need to reset the flag in either case.
	if new&mutexWoken == 0 {
		throw("sync: inconsistent mutex state")
	}
	new &^= mutexWoken // 清除唤醒标志位
}
```

虽然我们得出了 这段逻辑执行开始到执行最后，old和Mutex的state是一致的 这个结论，但是从

```go
new := old
```

开始old可以是任意状态的组合，由于有3个状态位，也就是最低的3位，分别是

```go
const (
	mutexLocked = 1 << iota // mutex is locked
	mutexWoken
	mutexStarving
)
```

mutexWoken这个标识位有点不一样，如果要对Mutex的state进行修改的话，需要经过  互斥锁根据上下文计算当前互斥锁最新的状态 这段逻辑 ，而在 这段逻辑中mutexWoken是会被清除的

```go
if awoke {
	// The goroutine has been woken from sleep,
	// so we need to reset the flag in either case.
	if new&mutexWoken == 0 {
		throw("sync: inconsistent mutex state")
	}
	new &^= mutexWoken // 清除唤醒标志位
}
```

所以state的mutexWoken字段到atomic.CompareAndSwapInt32()成功时只能是0

old的状态组合只有 4种情况

- mutexLocked & mutexStarving
- mutexLocked & 非mutexStarving
- 非mutexLocked & mutexStarving
- 非mutexLocked  & 非mutexStarving

相应的将这4种状态经过就算会得到new的状态

- mutexLocked & mutexStarving： 这种情况下mutexWaiter会+1
- mutexLocked & 非mutexStarving： 这种情况下mutexWaiter会+1
- 非mutexLocked & mutexStarving： 这种情况下mutexWaiter会+1
- 非mutexLocked  & 非mutexStarving：这种情况下会加锁成功

有了上面这个 状态表之后再来看atomic.CompareAndSwapInt32(&m.state, old, new)成功后的逻辑

```go

if atomic.CompareAndSwapInt32(&m.state, old, new) {
	if old&(mutexLocked|mutexStarving) == 0 {  // 这个就是上面状态表的第4种情况，说明加锁成功直接break
		break // locked the mutex with CAS
	}
	// 计算等待时间
	// If we were already waiting before, queue at the front of the queue.
	queueLifo := waitStartTime != 0
	if waitStartTime == 0 {
		waitStartTime = runtime_nanotime()
	}
	// 可以看到除了第4种情况下，其他情况mutexWaiter都会+1，也就意味着当前goroutine需要放入到阻塞队列，这里会阻塞知道锁被释放
	runtime_SemacquireMutex(&m.sema, queueLifo, 1)
	// 锁被释放后，继续从这里执行，如果等待时间操作了1ms，将当前goroutine的 starving 设置为 true
	starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
	old = m.state
	if old&mutexStarving != 0 { // Mutex处于饥饿状态
		// If this goroutine was woken and mutex is in starvation mode,
		// ownership was handed off to us but mutex is in somewhat
		// inconsistent state: mutexLocked is not set and we are still
		// accounted as waiter. Fix that.
		// 由于在饥饿模式中，互斥锁会直接交给等待队列最前面的 Goroutine。新的 Goroutine 在该状态下不能获取锁、也不会进入自旋状态，它们只会在队列的末尾等待。
		// 所以如果处于饥饿状态，当前Mutex的state不应该是mutexLocked或mutexWoken的
    // 而且一定会有mutexWaiter的goroutine，最少也是当前goroutine等待着
		if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
			throw("sync: inconsistent mutex state")
		}
		// 加锁并将mutexWaiter-1
		delta := int32(mutexLocked - 1<<mutexWaiterShift)
		if !starving || old>>mutexWaiterShift == 1 { // 如果当前是goroutine的 starving  为false，或者只有当前这个goroutine在等待，将Mutex的mutexStarving取消
			// Exit starvation mode.
			// Critical to do it here and consider wait time.
			// Starvation mode is so inefficient, that two goroutines
			// can go lock-step infinitely once they switch mutex
			// to starvation mode.
			delta -= mutexStarving  
		}
		// 完成加锁
		atomic.AddInt32(&m.state, delta)
		break
	}
	awoke = true
	iter = 0
}
```

# 解锁流程

Unlock()进行解锁

```go

func (m *Mutex) Unlock() {
	// Fast path: drop lock bit.
	new := atomic.AddInt32(&m.state, -mutexLocked) // 移除mutexLocked状态
	if new != 0 {
		// Outlined slow path to allow inlining the fast path.
		// To hide unlockSlow during tracing we skip one extra frame when tracing GoUnblock.
		m.unlockSlow(new)
	}
}
```

直接new := atomic.AddInt32(&m.state, -mutexLocked) 将mutexLocked状态移除，如果new==0的话，说明没有其他goroutine在等着，如果new ≠ 0 就需要处理其他goroutine

```go
func (m *Mutex) unlockSlow(new int32) {
	if (new+mutexLocked)&mutexLocked == 0 { // 检查状态是否正确
		throw("sync: unlock of unlocked mutex")
	}
	if new&mutexStarving == 0 { // 如果Mutex处于正常模式
		old := new
		for {
			// If there are no waiters or a goroutine has already
			// been woken or grabbed the lock, no need to wake anyone.
			// In starvation mode ownership is directly handed off from unlocking
			// goroutine to the next waiter. We are not part of this chain,
			// since we did not observe mutexStarving when we unlocked the mutex above.
			// So get off the way.
			// 如果没有 waiter，或者已经有在处理的情况，直接返回
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}
			// waiter 数减 1，mutexWoken 标志设置上，通过 CAS 更新 state 的值
			// Grab the right to wake someone.
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
				// 直接唤醒等待队列中的 waiter
				runtime_Semrelease(&m.sema, false, 1)
				return
			}
			old = m.state
		}
	} else { // 饥饿模式
		// Starving mode: handoff mutex ownership to the next waiter, and yield
		// our time slice so that the next waiter can start to run immediately.
		// Note: mutexLocked is not set, the waiter will set it after wakeup.
		// But mutex is still considered locked if mutexStarving is set,
		// so new coming goroutines won't acquire it.
		// 直接唤醒等待队列中的 waiter
		runtime_Semrelease(&m.sema, true, 1)
	}
}
```

在正常模式下，如果没有 waiter，或者mutexLocked、mutexStarving、mutexWoken有一个不为零说明已经有其他goroutine在处理了，直接返回；如果互斥锁存在等待者，那么通过runtime_Semrelease直接唤醒等待队列中的 waiter；

在饥饿模式，直接调用runtime_Semrelease方法将当前锁交给下一个正在尝试获取锁的等待者，等待者被唤醒后会得到锁。

# 参考资料

[Go 语言并发编程、同步原语与锁](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-sync-primitives/)

[多图详解Go的互斥锁Mutex - luozhiyun`s Blog](https://www.luozhiyun.com/archives/413)