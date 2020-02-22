# Netpoll
netpoll IO多路复用底层依赖于不同的框架，Linux下netpoll给予Epoll实现。


## 源码剖析

### netpoll 描述符

```
// netpoll 描述符
type pollDesc struct {
	link *pollDesc 

	// lock保护pollOpen、pollSetDeadline、pollUnblock和durationimpl操作。
	// pollReset、pollWait、pollWaitCanceled和runtime.netpollready(IO准备就绪通知),无需加锁操作。
	// 因此这些通知的closing、everr、rg、rd、wg和wd操作均无需无锁操作。

	fd      uintptr
	closing bool
	everr   bool    // 扫描错误发生时，用于标记
	user    uint32  // user settable cookie
	rseq    uintptr // 保护read过时时间
	rg      uintptr // 处于pdReady、pdWait G等待被读
	rt      timer   // read截止时间
	rd      int64   // read截止时间(int64)
	wseq    uintptr // 保护write过时时间
	wg      uintptr // 处于pdReady、pdWait G等待被写
	wt      timer   // write截止时间
	wd      int64   // write截止时间(int64)
}
```

```
// 存储pollDesc文件描述符，使用链表数据结构。
// 为保证可以从Epoll/Kqueu中获取就绪通知，PollDec对象必须处于type-stable。
// 使用seq检查过时通知，当deadlines被更改、描述符复用时，seq会增加。
type pollCache struct {	
	lock  mutex
	first *pollDesc
}
```

### gList

```
// A gList is a list of Gs linked through g.schedlink. A G can only be
// on one gQueue or gList at a time.
type gList struct {
	head guintptr
}
```
### netpoll
netpoll用于检查网络连接是否就绪，返回可运行Goroutine列表。
```
// delay < 0: 阻塞
// delay == 0: 非阻塞,轮训
// delay > 0: 阻塞时间
func netpoll(delay int64) gList {
	var waitms int32
	if delay < 0 {
		waitms = -1
	} else if delay == 0 {
		waitms = 0
	} else if delay < 1e6 {
		// 1ms
		waitms = 1
	} else if delay < 1e15 {
		// Max 1m
		waitms = int32(delay / 1e6)
	} else {
		// An arbitrary cap on how long to wait for a timer.
		// 1e9 ms == ~11.5 days.
		waitms = 1e9
	}


	var events [128]epollevent
retry:
	// epollwait，等待事件触发.
	// waitms超时时间(ms)。0会立即返回
	n := epollwait(epfd, &events[0], int32(len(events)), waitms)
	if n < 0 {
		if waitms > 0 {
			return gList{}
		}
		goto retry
	}

	var toRun gList
	for i := int32(0); i < n; i++ {
		ev := &events[i]

		// ...

		// 事件具体模式: 'r'、'w'、'r+w'
		var mode int32
		if ev.events&(_EPOLLIN|_EPOLLRDHUP|_EPOLLHUP|_EPOLLERR) != 0 {
			mode += 'r'
		}
		if ev.events&(_EPOLLOUT|_EPOLLHUP|_EPOLLERR) != 0 {
			mode += 'w'
		}

		if mode != 0 {
			pd := *(**pollDesc)(unsafe.Pointer(&ev.data))

			// ...

			// 已经就绪的事件,插入到待运行队列
			netpollready(&toRun, pd, mode)
		}
	}
	return toRun
}
```

### injectglist 

```
// 1. 将G状态更改为就绪状态(_Grunnable)，注入到调度器运行队列，等待调度运行。
// 2. 清空glist队列
func injectglist(glist *gList) {

	var n int
	for n = 0; !glist.empty(); n++ {
		// G状态更改为_Grunnable，并运行
		gp := glist.pop()
		casgstatus(gp, _Gwaiting, _Grunnable)
		globrunqput(gp)
	}

	// 如果当前程序中存在空闲处理器, 则通过runtime.startm函数启动新的线程执行这些任务。
	for ; n != 0 && sched.npidle != 0; n-- {
		startm(nil, false)
	}
	*glist = gList{}
}
```

### globrunqput

```
// 1. G放入全局队列，调度器可能被锁定。
// 2. 也许在STW,所以不允许写屏障
func globrunqput(gp *g) {
	sched.runq.pushBack(gp)
	sched.runqsize++
}
```

### epollwait

epollwait使用Plan9汇编通过系统调用(SYS_EPOLL_PWAIT)实现，具体系统调用编号可以参考如下:

[Linux_System_Call_Table_for_x86_64](https://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/)

```
TEXT runtime·epollwait(SB),NOSPLIT,$0
	// This uses pwait instead of wait, because Android O blocks wait.
	MOVL	epfd+0(FP), DI
	MOVQ	ev+8(FP), SI
	MOVL	nev+16(FP), DX
	MOVL	timeout+20(FP), R10
	MOVQ	$0, R8
	MOVL	$SYS_epoll_pwait, AX
	SYSCALL
	MOVL	AX, ret+24(FP)
	RET
```