---
title: "Timer源码阅读"
date: 2021-04-22T22:50:29+08:00
draft: false
original: true
categories: 
  - Golang
tags: 
  - Golang基础
---

# timer源码阅读

根据[6.3 计时器](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-timer/)中的描述，Golang Timer的设计经历了如下阶段：

1. Go 1.9 版本之前，所有的计时器由全局唯一的四叉堆维护；
2. Go 1.10 ~ 1.13，全局使用 64 个四叉堆维护全部的计时器，每个处理器（P）创建的计时器会由对应的四叉堆维护；
3. Go 1.14 版本之后，每个处理器单独管理计时器并通过网络轮询器触发；

- Go 1.9 版本之前由于使用全局的四叉堆，在多核情况下会出现锁竞争导致性能问题
- Go 1.10 ~ 1.13使用了64个四叉堆，有每个P来维护对应的四叉堆，相当于将锁的粒度减小，但是当timer在未到时间和到时间需要执行进行切换的时候，会发生P和M的绑定和解绑，尤其是当timer触发时间间隔比较小的情况下，会导致CPU占用过高，M/P切换的开销增加(TODO  为什么会发生P和M的绑定和解绑)
- Go 1.14 版本后每个P管理计时器四叉堆，由网络轮询器和调度器进行触发

我使用的是Go 1.16的版本进行分析

<!--more-->

# timer包的使用

主要分为2类，一次性触发的timer和多次触发的ticker

```go
func TestTimer(t *testing.T) {
	timer := time.NewTimer(time.Second)

	// for tm := range timer.C {
	// 	t.Log(tm)
	// 	timer.Reset(time.Second)
	// }

	var ch chan int
	for {
		select {
		case tm := <-timer.C:
			t.Log(tm)
			timer.Reset(time.Second)
		case <-ch:
		}
	}
}
```

```go
func TestAfter(t *testing.T) {
	var ch chan int
	select {
	case tm := <-time.After(time.Second):
		t.Log(tm)
	case <-ch:
	}
}
```

```go
func TestAfterFunc(t *testing.T) {
	var ch chan int
	timer := time.AfterFunc(time.Second, func() {
		t.Log("我执行了")
		ch <- 0
	})
	defer timer.Stop()
	<-ch
}
```

```go
func TestTicker(t *testing.T) {
	ticker := time.NewTicker(time.Second)
	var ch chan int
	for {
		select {
		case tm := <-ticker.C:
			t.Log(tm)
		case <-ch:
		}
	}
}
```

```go
func NewTicker(d Duration) *Ticker {
	if d <= 0 {
		panic(errors.New("non-positive interval for NewTicker"))
	}
	// Give the channel a 1-element time buffer.
	// If the client falls behind while reading, we drop ticks
	// on the floor until the client catches up.
	c := make(chan Time, 1)
	t := &Ticker{
		C: c,
		r: runtimeTimer{
			when:   when(d),
			period: int64(d),
			f:      sendTime,
			arg:    c,
		},
	}
	startTimer(&t.r)
	return t
}
```

```go

func TestTick(t *testing.T) {
	for tm := range time.Tick(time.Second) {
		t.Log(tm)
	}
}
```

需要注意的是，在for循环中使用的时候需要考虑是否会造成timer的泄漏；具体的示例分析可以参考 [Go 内存泄露之痛，这篇把 Go timer.After 问题根因讲透了！](https://mp.weixin.qq.com/s/KSBdPkkvonSES9Z9iggElg)

# 数据结构

```go
func NewTimer(d Duration) *Timer {
	c := make(chan Time, 1)
	t := &Timer{
		C: c,
		r: runtimeTimer{
			when: when(d),
			f:    sendTime,
			arg:  c,
		},
	}
	startTimer(&t.r)
	return t
}
```

可以看到NewTimer和NewTicker都会初始化runtimeTimer，差别在于Ticker会比Timer多了period参数。最后调用startTimer将timer添加到底层的最小四叉堆中

runtimeTimer和runtime/time.go#timer结构是保持一致的

```go
type timer struct {
	// If this timer is on a heap, which P's heap it is on.
	// puintptr rather than *p to match uintptr in the versions
	// of this struct defined in other packages.
	pp puintptr // 当前P的指针

	// Timer wakes up at when, and then at when+period, ... (period > 0 only)
	// each time calling f(arg, now) in the timer goroutine, so f must be
	// a well-behaved function and not block.
	//
	// when must be positive on an active timer.
	when   int64                      // 当前计时器被唤醒的时间
	period int64                      // 两次被唤醒的间隔
	f      func(interface{}, uintptr) // 每当计时器被唤醒时都会调用的函数
	arg    interface{}                // 计时器被唤醒时调用 f 传入的参数
	seq    uintptr

	// What to set the when field to in timerModifiedXX status.
	nextwhen int64 // 计时器处于 timerModifiedXX 状态时，用于设置 when 字段；

	// The status field holds one of the values below.
	status uint32 // 计时器的状态
}
```

# 添加timer

前面的startTimer方法其实是runtime/time.go中的startTimer方法，通过//go:linkname关联起来的

```go
// 把 t 添加到 timer 堆
// startTimer adds t to the timer heap.
//go:linkname startTimer time.startTimer
func startTimer(t *timer) {
	if raceenabled {
		racerelease(unsafe.Pointer(t))
	}
	addtimer(t)
}
```

继续调用addtimer方法

```go
// addtimer adds a timer to the current P.
// This should only be called with a newly created timer.
// That avoids the risk of changing the when field of a timer in some P's heap,
// which could cause the heap to become unsorted.
func addtimer(t *timer) {
	// when must be positive. A negative value will cause runtimer to
	// overflow during its delta calculation and never expire other runtime
	// timers. Zero will cause checkTimers to fail to notice the timer.
	if t.when <= 0 {
		throw("timer when must be positive")
	}
	if t.period < 0 {
		throw("timer period must be non-negative")
	}
	if t.status != timerNoStatus { // 添加新的timer必须是timerNoStatus
		throw("addtimer called with initialized timer")
	}
	t.status = timerWaiting

	when := t.when

	pp := getg().m.p.ptr()
	lock(&pp.timersLock)
	cleantimers(pp)
	doaddtimer(pp, t)
	unlock(&pp.timersLock)

	wakeNetPoller(when)
}
```

addtimer首先对参数进行了校验，timer的初始化status必须是timerNoStatus(计时器尚未设置状态)，然后将timer的status切换成timerWaiting(等待计时器启动)

然后调用cleantimers(pp)处理P中timers堆顶上已经取消(timerDeleted)或者时间发生改变(timerModifiedEarlier/timerModifiedLater的timer)，堆timers进行清理

```go
// 清理堆顶部的timer，与adjusttimers方法类似，只是adjusttimers会遍历搜索的timers
// 注意cleantimers清理的是堆顶部的timer，只要顶部是timerDeleted，timerModifiedEarlier/timerModifiedLater的timer都会处理
// 处理完后会调整堆，再处理堆顶部的timer，所以不只是处理1个timer，
// 当堆前面的timer是timerDeleted，timerModifiedEarlier/timerModifiedLater状态的时候都会进行处理
// adjusttimers不管是什么状态的timer，都会便利处理一遍
// cleantimers会出现下面2种状态的变化，也就是清除已经删除的，移动timer0
// timerDeleted -> timerRemoving -> timerRemoved
// timerModifiedEarlier/timerModifiedLater -> timerMoving -> timerWaiting
func cleantimers(pp *p) {
	gp := getg()
	for {
		if len(pp.timers) == 0 {
			return
		}

		// This loop can theoretically run for a while, and because
		// it is holding timersLock it cannot be preempted.
		// If someone is trying to preempt us, just return.
		// We can clean the timers later.
		if gp.preemptStop {
			return
		}

		t := pp.timers[0] // 堆顶，when最小，最早发生的timer
		if t.pp.ptr() != pp {
			throw("cleantimers: bad p")
		}
		switch s := atomic.Load(&t.status); s {
		case timerDeleted:
			// timerDeleted --> timerRemoving --> 从堆中删除timer --> timerRemoved
			if !atomic.Cas(&t.status, s, timerRemoving) {
				continue
			}
			dodeltimer0(pp)
			if !atomic.Cas(&t.status, timerRemoving, timerRemoved) {
				badTimer()
			}
			atomic.Xadd(&pp.deletedTimers, -1)
		case timerModifiedEarlier, timerModifiedLater: // TODO 如果modTimer将非timer0的when改成了比timer0更先触发的时候是怎么处理的
			// timerMoving --> 调整 timer 的时间 --> timerWaiting
			// 此时 timer 被调整为更早或更晚，将原先的 timer 进行删除，再重新添加
			if !atomic.Cas(&t.status, s, timerMoving) {
				continue
			}
			// Now we can change the when field.
			t.when = t.nextwhen
			// Move t to the right position.
			// 删除原来的
			dodeltimer0(pp)
			// 然后再重新添加
			doaddtimer(pp, t)
			if s == timerModifiedEarlier {
				atomic.Xadd(&pp.adjustTimers, -1) // 如果t0之前是timerModifiedEarlier，因为已经调整了t0，所以需要将adjustTimers减1
			}
			if !atomic.Cas(&t.status, timerMoving, timerWaiting) {
				badTimer()
			}
		default:
			// Head of timers does not need adjustment.
			return
		}
	}
}
```

cleantimers(pp *p)方法会循环处理堆顶部是timerDeleted，timerModifiedEarlier/timerModifiedLater的timer

- 如果是timerDeleted，说明timer已经取消了，从timers堆中删除，重新调整timers堆
- 如果是timerModifiedEarlier/timerModifiedLater，说明timer的时间发生改变，修改timer的when字段，删除之前的timer，重新添加新的timer

cleantimers(pp *p)方法只是对p中的timers堆做了一些清理调整的工作，真正添加是doaddtimer方法

```go
// doaddtimer adds t to the current P's heap.
// The caller must have locked the timers for pp.
func doaddtimer(pp *p, t *timer) {
	// Timers依赖network poller，确保netpoll经初始化了
	// Timers rely on the network poller, so make sure the poller
	// has started.
	if netpollInited == 0 {
		netpollGenericInit()
	}

	if t.pp != 0 { // 创建timer时没有绑定p，如果p存在的话属于异常情况
		throw("doaddtimer: P already set in timer")
	}
	t.pp.set(pp) // timer绑定到当前P的堆上
	i := len(pp.timers)
	pp.timers = append(pp.timers, t)
	siftupTimer(pp.timers, i) // 调整4叉堆
	if t == pp.timers[0] {    // 如果新加入的timer是当前p中最新触发的，将t.when保存到pp.timer0When
		atomic.Store64(&pp.timer0When, uint64(t.when))
	}
	atomic.Xadd(&pp.numTimers, 1)
}
```

doaddtimer方法中判断了netpoll是否初始化了，如果没有对其进行初始化，这里我还没有理解timer和netpoll之间的关系，作为todo，后续再补充

后面就是p的timer之间的绑定，添加到四叉堆，然后平衡四叉堆，这里就不细说了

doaddtimer方法返回后，回到addtimer方法会调用wakeNetPoller方法

```go
// wakeNetPoller wakes up the thread sleeping in the network poller if it isn't
// going to wake up before the when argument; or it wakes an idle P to service
// timers and the network poller if there isn't one already.
func wakeNetPoller(when int64) {
	if atomic.Load64(&sched.lastpoll) == 0 {
		// In findrunnable we ensure that when polling the pollUntil
		// field is either zero or the time to which the current
		// poll is expected to run. This can have a spurious wakeup
		// but should never miss a wakeup.
		pollerPollUntil := int64(atomic.Load64(&sched.pollUntil))
		if pollerPollUntil == 0 || pollerPollUntil > when { // 网络轮询器poll > timer的触发时间，立即唤醒netpoll
			netpollBreak()
		}
	} else {
		// There are no threads in the network poller, try to get
		// one there so it can handle new timers.
		if GOOS != "plan9" { // Temporary workaround - see issue #42303.
			wakep()
		}
	}
}
```

```go
// netpollBreak interrupts a kevent.
func netpollBreak() {
	if atomic.Cas(&netpollWakeSig, 0, 1) {
		for {
			var b byte
			n := write(netpollBreakWr, unsafe.Pointer(&b), 1)
			if n == 1 || n == -_EAGAIN {
				break
			}
			if n == -_EINTR {
				continue
			}
			println("runtime: netpollBreak write failed with", -n)
			throw("runtime: netpollBreak write failed")
		}
	}
}
```

wakeNetPoller方法其实就是向netpoll初始化的全局epfd文件描述符写入了数据（epfd和golang netpoll有关，想了解netpoll的请自行了解）。主要目的是为了唤醒netpoll

# 停止timer

可以通过Ticker#Stop和Timer#Stop停止timer

```go
// Stop prevents the Timer from firing.
// It returns true if the call stops the timer, false if the timer has already
// expired or been stopped.
// Stop does not close the channel, to prevent a read from the channel succeeding
// incorrectly.
//
// To ensure the channel is empty after a call to Stop, check the
// return value and drain the channel.
// For example, assuming the program has not received from t.C already:
//
// 	if !t.Stop() {
// 		<-t.C
// 	}
//
// This cannot be done concurrent to other receives from the Timer's
// channel or other calls to the Timer's Stop method.
//
// For a timer created with AfterFunc(d, f), if t.Stop returns false, then the timer
// has already expired and the function f has been started in its own goroutine;
// Stop does not wait for f to complete before returning.
// If the caller needs to know whether f is completed, it must coordinate
// with f explicitly.
func (t *Timer) Stop() bool {
	if t.r.f == nil {
		panic("time: Stop called on uninitialized Timer")
	}
	return stopTimer(&t.r)
}
```

```go
// Stop turns off a ticker. After Stop, no more ticks will be sent.
// Stop does not close the channel, to prevent a concurrent goroutine
// reading from the channel from seeing an erroneous "tick".
func (t *Ticker) Stop() {
	stopTimer(&t.r)
}
```

最后都是调用runtime.stopTimer方法；通过//go:linkname进行关联

```go
// stopTimer stops a timer.
// It reports whether t was stopped before being run.
//go:linkname stopTimer time.stopTimer
func stopTimer(t *timer) bool {
	return deltimer(t)
}
```

```go
// 返回的是这个timer在执行前被移除的，已经执行过了就返回false，还没有执行就返回true
// deltimer deletes the timer t. It may be on some other P, so we can't
// actually remove it from the timers heap. We can only mark it as deleted.
// It will be removed in due course by the P whose heap it is on.
// Reports whether the timer was removed before it was run.
func deltimer(t *timer) bool {
	for {
		switch s := atomic.Load(&t.status); s {
		case timerWaiting, timerModifiedLater: // timer还没启动或修改为更晚的时间
			// Prevent preemption while the timer is in timerModifying.
			// This could lead to a self-deadlock. See #38070.
			mp := acquirem()
			// timerWaiting/timerModifiedLater --> timerModifying --> timerDeleted
			if atomic.Cas(&t.status, s, timerModifying) { // TODO 为什么要先切换为timerModifying
				// Must fetch t.pp before changing status,
				// as cleantimers in another goroutine
				// can clear t.pp of a timerDeleted timer.
				tpp := t.pp.ptr()
				if !atomic.Cas(&t.status, timerModifying, timerDeleted) { // 置为timerDeleted状态
					badTimer()
				}
				releasem(mp)
				atomic.Xadd(&tpp.deletedTimers, 1)
				// Timer was not yet run.
				return true
			} else { // 修改为timerModifying失败，说明t的状态已经不再是timerWaiting, timerModifiedLater了
				releasem(mp) // 下一次再来处理
			}
		case timerModifiedEarlier:
			// Prevent preemption while the timer is in timerModifying.
			// This could lead to a self-deadlock. See #38070.
			mp := acquirem()
			// timerModifiedEarlier --> timerModifying --> timerDeleted
			if atomic.Cas(&t.status, s, timerModifying) {
				// Must fetch t.pp before setting status
				// to timerDeleted.
				tpp := t.pp.ptr()
				atomic.Xadd(&tpp.adjustTimers, -1) // timerModifiedEarlier的timer被stop了，所以需要将adjustTimers-1
				if !atomic.Cas(&t.status, timerModifying, timerDeleted) {
					badTimer()
				}
				releasem(mp)
				atomic.Xadd(&tpp.deletedTimers, 1)
				// Timer was not yet run.
				return true
			} else {
				releasem(mp) // 下一次再来处理
			}
		case timerDeleted, timerRemoving, timerRemoved:
			// Timer was already run.
			// Timer 已经运行
			return false
		case timerRunning, timerMoving:
			// 正在执行或被移动了，等待完成，下一次再来处理
			// The timer is being run or moved, by a different P.
			// Wait for it to complete.
			osyield()
		case timerNoStatus:
			// Removing timer that was never added or
			// has already been run. Also see issue 21874.
			return false
		case timerModifying:
			// 同时调用了deltimer，modtimer；等待其他调用完成，下一次再来处理
			// Simultaneous calls to deltimer and modtimer.
			// Wait for the other call to complete.
			osyield()
		default:
			badTimer()
		}
	}
}
```

从deltimer方法中可以看出，timer会发生如下的状态变化

- timerWaiting, timerModifiedLater → timerModifying → timerDeleted

    如果要停止的timer状态是timerWaiting, timerModifiedLater，说明timer还没有执行，将状态切换成timerModifying改变中，最后将状态切换成timerDeleted

- timerModifiedEarlier → timerModifying --> timerDeleted

    如果要停止的timer状态是timerModifiedEarlier，说明timer的时间被改变过，比如reset过，将状态切换成timerModifying改变中，最后将状态切换成timerDeleted

- timerDeleted, timerRemoving, timerRemoved → 什么都不做
- timerRunning, timerMoving → 等待操作完成
- timerNoStatus → 直接返回
- timerModifying → 等待操作完成

我在这里有2个问题

1. 为什么timer状态变化的时候需要需要先改为timerModifying然后再修改成最后的状态？

    答：首先声明这个只是我个人的理解可能会存在错误；在timer的status状态常量这有这么一段注释

    ```go
    // We don't permit calling addtimer/deltimer/modtimer/resettimer simultaneously,
    // but adjusttimers and runtimer can be called at the same time as any of those.
    ```

    为了保证addtimer/deltimer/modtimer/resettimer不能被同时调用，所以需要timerModifying这个状态

2. deltimer并没有从 四叉堆中删除timer，只是将timer的状态切换成timerDeleted，这个是为什么？

    这个在deltimer的注释上已经说明了

    ```go
    // deltimer deletes the timer t. It may be on some other P, so we can't
    // actually remove it from the timers heap. We can only mark it as deleted.
    // It will be removed in due course by the P whose heap it is on.
    ```

    deltimer删除的timer可能在其他P上，以为调度循环的 时候不仅会从其他P上偷G，还会偷timer，所以只是对timer进行标记，在timer所在的P中，通过 cleantimers/adjusttimers等方法来真正从堆中删除

# 其他timer的方法

分析了2个timer的方法后，就不再逐个看其他的方法了，大概都差不多，都是对timers堆中的timer状态进行修改，timers的调整等

## 修改timer

```go
// modtimer modifies an existing timer.
// This is called by the netpoll code or time.Ticker.Reset or time.Timer.Reset.
// Reports whether the timer was modified before it was run.
func modtimer(t *timer, when, period int64, f func(interface{}, uintptr), arg interface{}, seq uintptr) bool {
	if when <= 0 {
		throw("timer when must be positive")
	}
	if period < 0 {
		throw("timer period must be non-negative")
	}

	status := uint32(timerNoStatus)
	wasRemoved := false
	var pending bool
	var mp *m
loop:
	for {
		switch status = atomic.Load(&t.status); status {
		case timerWaiting, timerModifiedEarlier, timerModifiedLater:
			// Prevent preemption while the timer is in timerModifying.
			// This could lead to a self-deadlock. See #38070.
			mp = acquirem()
			// timerWaiting, timerModifiedEarlier, timerModifiedLater --> timerModifying
			if atomic.Cas(&t.status, status, timerModifying) {
				pending = true // timer not yet run
				break loop
			}
			releasem(mp)
		case timerNoStatus, timerRemoved:
			// Prevent preemption while the timer is in timerModifying.
			// This could lead to a self-deadlock. See #38070.
			mp = acquirem()

			// Timer was already run and t is no longer in a heap.
			// Act like addtimer.
			// timerNoStatus, timerRemoved --> timerModifying
			if atomic.Cas(&t.status, status, timerModifying) {
				wasRemoved = true
				pending = false // timer already run or stopped
				break loop
			}
			releasem(mp)
		case timerDeleted:
			// Prevent preemption while the timer is in timerModifying.
			// This could lead to a self-deadlock. See #38070.
			mp = acquirem()
			// timerDeleted --> timerModifying
			if atomic.Cas(&t.status, status, timerModifying) {
				atomic.Xadd(&t.pp.ptr().deletedTimers, -1)
				pending = false // timer already stopped
				break loop
			}
			releasem(mp)
		case timerRunning, timerRemoving, timerMoving:
			// The timer is being run or moved, by a different P.
			// Wait for it to complete.
			osyield() // 等待状态改变
		case timerModifying:
			// Multiple simultaneous calls to modtimer.
			// Wait for the other call to complete.
			osyield() // 等待状态改变
		default:
			badTimer()
		}
	}

	t.period = period
	t.f = f
	t.arg = arg
	t.seq = seq

	if wasRemoved {
		t.when = when
		pp := getg().m.p.ptr()
		lock(&pp.timersLock)
		doaddtimer(pp, t)
		unlock(&pp.timersLock)
		if !atomic.Cas(&t.status, timerModifying, timerWaiting) {
			badTimer()
		}
		releasem(mp)
		wakeNetPoller(when)
	} else {
		// The timer is in some other P's heap, so we can't change
		// the when field. If we did, the other P's heap would
		// be out of order. So we put the new when value in the
		// nextwhen field, and let the other P set the when field
		// when it is prepared to resort the heap.
		t.nextwhen = when

		newStatus := uint32(timerModifiedLater)
		if when < t.when {
			newStatus = timerModifiedEarlier
		}

		tpp := t.pp.ptr()

		// Update the adjustTimers field.  Subtract one if we
		// are removing a timerModifiedEarlier, add one if we
		// are adding a timerModifiedEarlier.
		adjust := int32(0)
		if status == timerModifiedEarlier {
			adjust--
		}
		if newStatus == timerModifiedEarlier {
			adjust++
			updateTimerModifiedEarliest(tpp, when)
		}
		if adjust != 0 {
			atomic.Xadd(&tpp.adjustTimers, adjust)
		}

		// Set the new status of the timer.
		if !atomic.Cas(&t.status, timerModifying, newStatus) {
			badTimer()
		}
		releasem(mp)

		// If the new status is earlier, wake up the poller.
		if newStatus == timerModifiedEarlier {
			wakeNetPoller(when)
		}
	}

	return pending
}
```

## 调整timer

```go
// 与cleantimers类似，只是 cleantimers 只处理队列头部的timer
// adjusttimers looks through the timers in the current P's heap for
// any timers that have been modified to run earlier, and puts them in
// the correct place in the heap. While looking for those timers,
// it also moves timers that have been modified to run later,
// and removes deleted timers. The caller must have locked the timers for pp.
func adjusttimers(pp *p, now int64) {
	if atomic.Load(&pp.adjustTimers) == 0 {
		if verifyTimers {
			verifyTimerHeap(pp)
		}
		// There are no timers to adjust, so it is safe to clear
		// timerModifiedEarliest. Do so in case it is stale.
		// Everything will work if we don't do this,
		// but clearing here may save future calls to adjusttimers.
		atomic.Store64(&pp.timerModifiedEarliest, 0)
		return
	}

	// If we haven't yet reached the time of the first timerModifiedEarlier
	// timer, don't do anything. This speeds up programs that adjust
	// a lot of timers back and forth if the timers rarely expire.
	// We'll postpone looking through all the adjusted timers until
	// one would actually expire.
	if first := atomic.Load64(&pp.timerModifiedEarliest); first != 0 {
		if int64(first) > now {
			if verifyTimers {
				verifyTimerHeap(pp)
			}
			return
		}

		// We are going to clear all timerModifiedEarlier timers.
		atomic.Store64(&pp.timerModifiedEarliest, 0)
	}

	var moved []*timer
loop:
	for i := 0; i < len(pp.timers); i++ {
		t := pp.timers[i]
		if t.pp.ptr() != pp {
			throw("adjusttimers: bad p")
		}
		switch s := atomic.Load(&t.status); s {
		case timerDeleted:
			if atomic.Cas(&t.status, s, timerRemoving) {
				dodeltimer(pp, i)
				if !atomic.Cas(&t.status, timerRemoving, timerRemoved) {
					badTimer()
				}
				atomic.Xadd(&pp.deletedTimers, -1)
				// Look at this heap position again.
				i--
			}
		case timerModifiedEarlier, timerModifiedLater:
			if atomic.Cas(&t.status, s, timerMoving) {
				// Now we can change the when field.
				t.when = t.nextwhen
				// Take t off the heap, and hold onto it.
				// We don't add it back yet because the
				// heap manipulation could cause our
				// loop to skip some other timer.
				dodeltimer(pp, i)
				moved = append(moved, t)
				if s == timerModifiedEarlier {
					if n := atomic.Xadd(&pp.adjustTimers, -1); int32(n) <= 0 {
						break loop
					}
				}
				// Look at this heap position again.
				i--
			}
		case timerNoStatus, timerRunning, timerRemoving, timerRemoved, timerMoving:
			badTimer()
		case timerWaiting:
			// OK, nothing to do.
		case timerModifying:
			// Check again after modification is complete.
			osyield()
			i--
		default:
			badTimer()
		}
	}

	if len(moved) > 0 {
		addAdjustedTimers(pp, moved)
	}

	if verifyTimers {
		verifyTimerHeap(pp)
	}
}
```

## 运行timer

```go
// runtimer 检查timers四叉堆顶部的timer
// runtimer examines the first timer in timers. If it is ready based on now,
// it runs the timer and removes or updates it.
// Returns 0 if it ran a timer, -1 if there are no more timers, or the time
// when the first timer should run.
// The caller must have locked the timers for pp.
// If a timer is run, this will temporarily unlock the timers.
//go:systemstack
func runtimer(pp *p, now int64) int64 {
	for {
		t := pp.timers[0]
		if t.pp.ptr() != pp {
			throw("runtimer: bad p")
		}
		switch s := atomic.Load(&t.status); s {
		case timerWaiting:
			if t.when > now { // 还没到时间执行
				// Not ready to run.
				return t.when
			}
			// 该执行这个timer了
			if !atomic.Cas(&t.status, s, timerRunning) {
				continue
			}
			// Note that runOneTimer may temporarily unlock
			// pp.timersLock.
			runOneTimer(pp, t, now)
			return 0

		case timerDeleted: // 删除已经执行了的timer
			if !atomic.Cas(&t.status, s, timerRemoving) {
				continue
			}
			dodeltimer0(pp)
			if !atomic.Cas(&t.status, timerRemoving, timerRemoved) {
				badTimer()
			}
			atomic.Xadd(&pp.deletedTimers, -1)
			if len(pp.timers) == 0 {
				return -1
			}

		case timerModifiedEarlier, timerModifiedLater: // 调整timerModifiedEarlier, timerModifiedLater timer的时间
			if !atomic.Cas(&t.status, s, timerMoving) {
				continue
			}
			t.when = t.nextwhen
			dodeltimer0(pp)
			doaddtimer(pp, t)
			if s == timerModifiedEarlier {
				atomic.Xadd(&pp.adjustTimers, -1)
			}
			if !atomic.Cas(&t.status, timerMoving, timerWaiting) {
				badTimer()
			}

		case timerModifying:
			// Wait for modification to complete.
			osyield() // 等到其他操作结束

		case timerNoStatus, timerRemoved:
			// Should not see a new or inactive timer on the heap.
			badTimer()
		case timerRunning, timerRemoving, timerMoving:
			// These should only be set when timers are locked,
			// and we didn't do it.
			badTimer()
		default:
			badTimer()
		}
	}
}
```

# 触发timer

前面介绍的都是将 timer加入到 堆中，从堆中删除这些，那么timer时间到了，是怎么触发的呢？

触发timer一定会执行前面所说的runtimer方法，可以发现runtimer是在checkTimers方法中被调用的

```go
// checkTimers runs any timers for the P that are ready.
// If now is not 0 it is the current time.
// It returns the current time or 0 if it is not known,
// and the time when the next timer should run or 0 if there is no next timer,
// and reports whether it ran any timers.
// If the time when the next timer should run is not 0,
// it is always larger than the returned time.
// We pass now in and out to avoid extra calls of nanotime.
//go:yeswritebarrierrec
func checkTimers(pp *p, now int64) (rnow, pollUntil int64, ran bool) {
	// If it's not yet time for the first timer, or the first adjusted
	// timer, then there is nothing to do.
	next := int64(atomic.Load64(&pp.timer0When))
	nextAdj := int64(atomic.Load64(&pp.timerModifiedEarliest))
	if next == 0 || (nextAdj != 0 && nextAdj < next) {
		next = nextAdj
	}

	if next == 0 { // 没有timer需要执行和调整
		// No timers to run or adjust.
		return now, 0, false
	}

	if now == 0 {
		now = nanotime()
	}
	if now < next { // 最快的 timer还没到 执行的时间
		// Next timer is not ready to run, but keep going
		// if we would clear deleted timers.
		// This corresponds to the condition below where
		// we decide whether to call clearDeletedTimers.
		if pp != getg().m.p.ptr() || int(atomic.Load(&pp.deletedTimers)) <= int(atomic.Load(&pp.numTimers)/4) {
			return now, next, false
		}
	}

	lock(&pp.timersLock)

	if len(pp.timers) > 0 {
		adjusttimers(pp, now)    // 删除已经执行的timer，调整timerModifiedEarlier 和 timerModifiedLater 的计时器的时间
		for len(pp.timers) > 0 { // 执行所有到期的timer
			// Note that runtimer may temporarily unlock
			// pp.timersLock.
			if tw := runtimer(pp, now); tw != 0 {
				if tw > 0 {
					pollUntil = tw
				}
				break
			}
			ran = true
		}
	}

	// If this is the local P, and there are a lot of deleted timers,
	// clear them out. We only do this for the local P to reduce
	// lock contention on timersLock.
	// 当前 Goroutine 的处理器和传入的处理器相同,并且处理器中删除的计时器是堆中计时器的 1/4 以上，
	if pp == getg().m.p.ptr() && int(atomic.Load(&pp.deletedTimers)) > len(pp.timers)/4 {
		clearDeletedTimers(pp)
	}

	unlock(&pp.timersLock)

	return now, pollUntil, ran
}
```

而checkTimers在findrunnable和schedule中被调用，而这2个方法都是runtime调度会执行的方法（PS：runtime调度也是一个很重要的知识点，有兴趣的可以自行了解）

除了runtime调度时会执行timer外，系统监控sysmon也会执行timer，其实这里我没有理解，所以这里直接用draveness大佬文章中的说明

系统监控函数 runtime.sysmon 也可能会触发函数的计时器，下面的代码片段中省略了大量与计时器无关的代码：

```go
func sysmon() {
	...
	for {
		...
		now := nanotime()
		next, _ := timeSleepUntil()
		...
		lastpoll := int64(atomic.Load64(&sched.lastpoll))
		if netpollinited() && lastpoll != 0 && lastpoll+10*1000*1000 < now {
			atomic.Cas64(&sched.lastpoll, uint64(lastpoll), uint64(now))
			list := netpoll(0)
			if !list.empty() {
				incidlelocked(-1)
				injectglist(&list)
				incidlelocked(1)
			}
		}
		if next < now {
			startm(nil, false)
		}
		...
}
```

1. 调用 `[runtime.timeSleepUntil](https://draveness.me/golang/tree/runtime.timeSleepUntil)` 获取计时器的到期时间以及持有该计时器的堆；
2. 如果超过 10ms 的时间没有轮询，调用 `[runtime.netpoll](https://draveness.me/golang/tree/runtime.netpoll)` 轮询网络；
3. 如果当前有应该运行的计时器没有执行，可能存在无法被抢占的处理器，这时我们应该启动新的线程处理计时器；

在上述过程中 `[runtime.timeSleepUntil](https://draveness.me/golang/tree/runtime.timeSleepUntil)` 会遍历运行时的全部处理器并查找下一个需要执行的计时器。

# 遗留问题

最后是我还存在的问题

1. sysmon中为什么会触发timer
2. addtimer方法中调用了wakeNetPoller(when)方法唤醒netpoll，但是netpoll()方法中对netpollBreakRd的处理并没有发现与timer有啥关系

    ```go
    // netpoll checks for ready network connections.
    // Returns list of goroutines that become runnable.
    // delay < 0: blocks indefinitely
    // delay == 0: does not block, just polls
    // delay > 0: block for up to that many nanoseconds
    // delay < 0 无限block等待
    // delay == 0 不会block
    // delay block 最多delay时间
    // runtime.netpoll 返回的 Goroutine 列表都会被 runtime.injectglist 注入到处理器或者全局的运行队列上。
    // 因为系统监控 Goroutine 直接运行在线程上，所以它获取的 Goroutine 列表会直接加入全局的运行队列，
    // 其他 Goroutine 获取的列表都会加入 Goroutine 所在处理器的运行队列上。
    func netpoll(delay int64) gList {
    	if epfd == -1 { // 没有epfd 相当于netpoll没有初始化
    		return gList{}
    	}
    	var waitms int32
    	if delay < 0 {
    		waitms = -1
    	} else if delay == 0 {
    		waitms = 0
    	} else if delay < 1e6 {
    		waitms = 1
    	} else if delay < 1e15 {
    		waitms = int32(delay / 1e6)
    	} else {
    		// An arbitrary cap on how long to wait for a timer.
    		// 1e9 ms == ~11.5 days.
    		waitms = 1e9
    	}
    	var events [128]epollevent
    retry:
    	// 等待文件描述符转换成可读或者可写
    	n := epollwait(epfd, &events[0], int32(len(events)), waitms)
    	if n < 0 { // 如果返回了负值，可能会返回空的 Goroutine 列表或者重新调用 epollwait 陷入等待：
    		if n != -_EINTR {
    			println("runtime: epollwait on fd", epfd, "failed with", -n)
    			throw("runtime: netpoll failed")
    		}
    		// If a timed sleep was interrupted, just return to
    		// recalculate how long we should sleep now.
    		if waitms > 0 {
    			return gList{}
    		}
    		goto retry
    	}
    	// 当 epollwait 系统调用返回的值大于 0 时，意味着被监控的文件描述符出现了待处理的事件
    	var toRun gList
    	for i := int32(0); i < n; i++ {
    		ev := &events[i]
    		if ev.events == 0 {
    			continue
    		}

    		// runtime.netpollBreak 触发的事件
    		if *(**uintptr)(unsafe.Pointer(&ev.data)) == &netpollBreakRd {
    			if ev.events != _EPOLLIN {
    				println("runtime: netpoll: break fd ready for", ev.events)
    				throw("runtime: netpoll: break fd ready for something unexpected")
    			}
    			if delay != 0 {
    				// netpollBreak could be picked up by a
    				// nonblocking poll. Only read the byte
    				// if blocking.
    				var tmp [16]byte
    				read(int32(netpollBreakRd), noescape(unsafe.Pointer(&tmp[0])), int32(len(tmp)))
    				atomic.Store(&netpollWakeSig, 0)
    			}
    			continue
    		}

    		// 另一种是其他文件描述符的正常读写事件
    		var mode int32
    		if ev.events&(_EPOLLIN|_EPOLLRDHUP|_EPOLLHUP|_EPOLLERR) != 0 {
    			mode += 'r'
    		}
    		if ev.events&(_EPOLLOUT|_EPOLLHUP|_EPOLLERR) != 0 {
    			mode += 'w'
    		}
    		if mode != 0 {
    			pd := *(**pollDesc)(unsafe.Pointer(&ev.data))
    			pd.everr = false
    			if ev.events == _EPOLLERR {
    				pd.everr = true
    			}
    			netpollready(&toRun, pd, mode)
    		}
    	}
    	return toRun
    }
    ```

    draveness大佬文章的评论中也有人提到这个疑问，但是还是未能理解，我也加入 了[讨论](https://github.com/draveness/blog-comments/issues/152#issuecomment-824642345)，期待后续的解答

# 参考资料

- [6.3 计时器](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-timer/)
- [Go中定时器实现原理及源码解析](https://www.luozhiyun.com/archives/458)
- [难以驾驭的 Go timer，一文带你参透计时器的奥秘](https://mp.weixin.qq.com/s/gxX-q2EvgWZEWe-deRITSw)
- [go1.14基于netpoll优化timer定时器实现原理](http://xiaorui.cc/archives/6483)
- [https://www.youtube.com/watch?v=XJx0eTP-y9I](https://www.youtube.com/watch?v=XJx0eTP-y9I)
