# retake

retake函数会遍历运行中的全部Process，对于长时间占用Process资源进行抢占，以实现Process尽可能公平调度。主要抢占_Prunning和_Psyscall中的process，每个Process都存储一个sysmontick结构体。

* _Prunning: Prunning抢占通常是由于一些类似死循环的计算逻辑引起的。
* _Psyscall: Psyscall抢占通常是由于阻塞性系统调用引起的，比如磁盘IO、CGO等。

## retake使用的全局环境变量

```
// runtime/runtime2.go:
var (
    allp       []*p         // len(allp) == runtimie.GOMAXPROCESS
    allpLock   mutex        // 保护allp的读和写
)

// runtime/proc.go:
// 距离上次抢占P的时间,超过该时间则执行抢占
const forcePreemptNS = 10 * 1000 * 1000 // 10ms

// runtime/stack.go:
// stackPreempt随意的一个内存地址，该值大于整个stack sp值。主要用于检查栈帧溢出时使用
stackPreempt = uintptrMask & -131


// runtime/signal_amd64.go:
const pushCallSupported = true

// runtime/signal_unix.go:
const preemptMSupported = pushCallSupported

// runtime/runtime1.go:
var debug struct {
	// ...

    // 0 
	asyncpreemptoff    int32        
}

// runtime/defs_linux_amd64.go:
const sigPreempt = _SIGURG

// runtime/defs_linux_amd64.go:
const _SIGURG    = 0x17         // SIGURG值，23
```

## reteke源码剖析

```
func retake(now int64) uint32 {
	n := 0
	
	for i := 0; i < len(allp); i++ {
		_p_ := allp[i]

		pd := &_p_.sysmontick  	// sysmon上次扫描记录
		s := _p_.status
		sysretake := false

		// _Prunning抢占
		if s == _Prunning || s == _Psyscall {
			// Preempt G if it's running for too long.
			t := int64(_p_.schedtick)
			if int64(pd.schedtick) != t {
				pd.schedtick = uint32(t)
				pd.schedwhen = now
			} else if pd.schedwhen+forcePreemptNS <= now {
				// 当Process处于_Prunning或者_Psyscall状态时,如果距上一次调度时间过去了10ms，则通过preemptone函数抢占当前处理器。
				// 如果是syscall,则preemptone不会有效，因为没有MP绑定
				preemptone(_p_)

				// 开启Psyscall抢占
				sysretake = true
			}
		}

		// _Psyscall抢占
		if s == _Psyscall {
			// Retake P from syscall if it's there for more than 1 sysmon tick (at least 20us).
			t := int64(_p_.syscalltick)

			// ...
			if atomic.Cas(&_p_.status, s, _Pidle) {
				n++
				_p_.syscalltick++
				handoffp(_p_)
			}

			// ...
		}
	}

	// 抢占到Process的数量
	return uint32(n)
}
```

### _Prunning抢占
_Prunning主要将长时间运行的G任务发出抢占调度。 

#### sysmontick

```
type sysmontick struct {
	schedtick   uint32		// Process的调度次数
	schedwhen   int64		// Process上次调度事件
	syscalltick uint32		// 系统调用次数
	syscallwhen int64		// 系统调用时间
}
```

#### preempton
```
// 抢占当前Process
func preemptone(_p_ *p) bool {
	mp := _p_.m.ptr()

	gp := mp.curg

	// ...

	gp.preempt = true

	// ...

	// 每次函数调用，都需要通过gp.stackguard0检查stack是否溢出。
	// stackPreempt指针大于整个stack sp值,则会触发栈溢出。
	// 尽可能保证Process被抢占成功
	gp.stackguard0 = stackPreempt

	// 1.14版本新增
	// 发送SIGURG信号，异步抢占
	if preemptMSupported && debug.asyncpreemptoff == 0 {
		_p_.preempt = true
		preemptM(mp)
	}

	return true
}
```

#### preemptM

```
// preemptM向mp发送一个抢占请求, 异步处理
// runtime/signal_unix.go:
func preemptM(mp *m) {

	// sigPreempt -> SIGURG
	signalM(mp, sigPreempt)
}
```

#### signalM

```
// runtime/os_linux.go:
// 向MP发送一个信号
func signalM(mp *m, sig int) {
	tgkill(getpid(), int(mp.procid), sig)
}
```

### _Psyscall抢占
Psyscall抢占通常是由于阻塞性系统调用引起的，比如磁盘IO、CGO等。

#### handoffp

```
// 抢占Process
func handoffp(_p_ *p) {

	// 如果本地P队列有任务运行,则立即执行
	if !runqempty(_p_) || sched.runqsize != 0 {
		startm(_p_, false)
		return
	}
	// 如果有GC任务，则立即开始
	if gcBlackenEnabled != 0 && gcMarkWorkAvailable(_p_) {
		startm(_p_, false)
		return
	}

	// 如果本地没有任务，检查是否有自旋/空闲M。如果有,则立即执行
	if atomic.Load(&sched.nmspinning)+atomic.Load(&sched.npidle) == 0 && atomic.Cas(&sched.nmspinning, 0, 1) { // TODO: fast atomic
		startm(_p_, true)
		return
	}

	// GC, STW
	if sched.gcwaiting != 0 {
		_p_.status = _Pgcstop
		sched.stopwait--
		if sched.stopwait == 0 {
			notewakeup(&sched.stopnote)
		}
		return
	}

	// 垃圾回收算法通常都有个阶段暂停所有线程对内存对象引用关系网络的更新，这个机制称为Safepoint。
	// _Pidle或_Psyscall进行状态转换时,都需要检查runSafePointFn。
	// runSafePointFn抢占P
	if _p_.runSafePointFn != 0 && atomic.Cas(&_p_.runSafePointFn, 1, 0) {
		sched.safePointFn(_p_)
		sched.safePointWait--
		if sched.safePointWait == 0 {
			notewakeup(&sched.safePointNote)
		}
	}

	// 全局队列不为空,则立即执行
	if sched.runqsize != 0 {
		startm(_p_, false)
		return
	}

	
	// 如果这是最后一个P，并且没有任何人正在polling network。
	// 则唤醒一个M执行polling netowrk。
	if sched.npidle == uint32(gomaxprocs-1) && atomic.Load64(&sched.lastpoll) != 0 {
		startm(_p_, false)
		return
	}

	// 1.14版本的优化
	// 如果P中有下次唤醒Netpoller时间,则在when时唤醒netpoller休眠的线程
	if when := nobarrierWakeTime(_p_); when != 0 {
		wakeNetPoller(when)
	}

	// P抢占成功,将P放入_Pidle列表
	pidleput(_p_)
}
```

#### nobarrierWakeTime

```
// 1. 查看P的计时器,并返回唤醒Netpoller的时间。如果没有计时器，则返回0。
// 2. Netpoller M在休眠之前再次尝试唤醒并调整计数器。
func nobarrierWakeTime(pp *p) int64 {
	if atomic.Load(&pp.adjustTimers) > 0 {
		return nanotime()
	} else {
		return int64(atomic.Load64(&pp.timer0When))
	}
}
```

#### pidleput

```
// 将P放入_Pidle列表
func pidleput(_p_ *p) {
	_p_.link = sched.pidle
	sched.pidle.set(_p_)
	atomic.Xadd(&sched.npidle, 1)
}
```