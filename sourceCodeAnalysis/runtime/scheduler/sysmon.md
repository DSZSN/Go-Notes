# sysmon
sysmon是一个由runtime启动的M，也叫监控线程，它无需P也可以运行，它每20us~10ms唤醒一次。本文给予Go1.14版本进行剖析,Go1.14调度器主要添加了一个SIGURG信号,处理之前for{ // loop }循环无法抢占调度的问题。

sysmon主要执行:

* 处理大于10ms未执行的netpoll文件描述符，将所有处于就绪的G加入到P全局队列，等待被执行。
* 超过2min未执行的GC，将强制执行。
* 将长时间运行的G任务发出抢占调度。
* 回收因syscall长时间阻塞的P。

## 源码剖析

```
// runtime/proc.go:sysmon
func sysmon() {
	// ...

	lasttrace := int64(0)
	idle := 0 // how many cycles in succession we had not wokeup somebody
	delay := uint32(0)
	for {
		if idle == 0 { // start with 20us sleep...
			delay = 20
		} else if idle > 50 { // start doubling the sleep after 1ms...
			delay *= 2
		}
		if delay > 10*1000 { // up to 10ms
			delay = 10 * 1000
		}

        // 休眠delay时间，开始执行sysmon操作
		usleep(delay)

		// 当前时间
		now := nanotime()

		// 下一次唤醒时间
		next, _ := timeSleepUntil()


		// GC STW 或者 所有P处于空闲, 则sysmon休眠
		// Notes: 所有P处于空闲状态, sysmon执行没有意义。
		if debug.schedtrace <= 0 && (sched.gcwaiting != 0 || atomic.Load(&sched.npidle) == uint32(gomaxprocs)) {

			if atomic.Load(&sched.gcwaiting) != 0 || atomic.Load(&sched.npidle) == uint32(gomaxprocs) {
				if next > now {
					// sysmon进入休眠
					atomic.Store(&sched.sysmonwait, 1)

					// 尽可能短时间发起抢占
					sleep := forcegcperiod / 2
					if next-now < sleep {
						sleep = next - now
					}

					// ...

					// 休眠,等待sleep纳秒后被唤醒
					notetsleep(&sched.sysmonnote, sleep)
					if shouldRelax {
						osRelax(false)
					}

					now = nanotime()
					next, _ = timeSleepUntil()

					// Sysmon 退出休眠
					atomic.Store(&sched.sysmonwait, 0)

					// noteclear等待所有notewakeup执行完后被执行
					noteclear(&sched.sysmonnote)
				}
				idle = 0
				delay = 20
			}
		}

		// CGO ...

		// schedinit初始化时,设置了sched.lastpoll值:
		// sched.lastpoll = uint64(nanotime())
		// 大于10ms未处理的netpoll 文件描述符,将所有处于就绪状态的G加入到全局队列,等待被调度执行
		lastpoll := int64(atomic.Load64(&sched.lastpoll))
		if netpollinited() && lastpoll != 0 && lastpoll+10*1000*1000 < now {
			atomic.Cas64(&sched.lastpoll, uint64(lastpoll), uint64(now))

			// 非阻塞, 就绪gList队列
            // netpoll源码剖析，请查看netpoll.md。
			list := netpoll(0)
			if !list.empty() {
				// G放入全局队列，调度运行
				injectglist(&list)
			}
		}

		if next < now {
			// 1.14版本优化:
			// 可能由于P不能被抢占,导致计数器无法被执行。启动一个新的M运行它们
			// Eg:
			// func {
			//
			// 		go func(){
			//			panic("already call")
			// 		}()
			//	
			//		for {
			//
			//		}
			//
			//
			// }
			startm(nil, false)
		}


		// 1. 抢占_Prunning和_Psyscall中的Process，retake会遍历运行中的全部处理器,每个Process都存储一个sysmontick结构体。
		// 2. Psyscall抢占通常是由于阻塞性系统调用引起的，比如磁盘IO、CGO等。
		// 3. Prunning抢占通常是由于一些类似死循环的计算逻辑引起的。
        // P抢占调度请查看retake.md
		if retake(now) != 0 {
			// 成功抢占到P
			idle = 0
		} else {
			// 抢占P失败
			idle++
		}

		// 如果超过2min没有触发GC，则强制执行GC
		if t := (gcTrigger{kind: gcTriggerTime, now: now}); t.test() && atomic.Load(&forcegc.idle) != 0 {

			forcegc.idle = 0
			var list gList
			list.push(forcegc.g)
			injectglist(&list)

		}

		if debug.schedtrace > 0 && lasttrace+int64(debug.schedtrace)*1000000 <= now {
			lasttrace = now
			schedtrace(debug.scheddetail > 0)
		}
	}
}
```

## 信号
对于Linux来说，实际信号是软中断，许多重要的程序都需要处理信号。信号，为 Linux 提供了一种处理异步事件的方法。比如，终端用户输入了 ctrl+c 来中断程序，会通过信号机制停止一个程序。

每个信号都有一个名字和编号，这些名字都以“SIG”开头，例如“SIGIO ”、“SIGCHLD”等等。
信号定义在signal.h头文件中，信号名都定义为正整数。

具体的信号名称可以使用kill -l来查看信号的名字以及序号，信号是从1开始编号的，不存在0号信号。kill对于信号0又特殊的应用。

```
 1) SIGHUP	 2) SIGINT	 3) SIGQUIT	 4) SIGILL	 5) SIGTRAP
 6) SIGABRT	 7) SIGBUS	 8) SIGFPE	 9) SIGKILL	10) SIGUSR1
11) SIGSEGV	12) SIGUSR2	13) SIGPIPE	14) SIGALRM	15) SIGTERM
16) SIGSTKFLT	17) SIGCHLD	18) SIGCONT	19) SIGSTOP	20) SIGTSTP
21) SIGTTIN	22) SIGTTOU	23) SIGURG	24) SIGXCPU	25) SIGXFSZ
26) SIGVTALRM	27) SIGPROF	28) SIGWINCH	29) SIGIO	30) SIGPWR
31) SIGSYS	34) SIGRTMIN	35) SIGRTMIN+1	36) SIGRTMIN+2	37) SIGRTMIN+3
38) SIGRTMIN+4	39) SIGRTMIN+5	40) SIGRTMIN+6	41) SIGRTMIN+7	42) SIGRTMIN+8
43) SIGRTMIN+9	44) SIGRTMIN+10	45) SIGRTMIN+11	46) SIGRTMIN+12	47) SIGRTMIN+13
48) SIGRTMIN+14	49) SIGRTMIN+15	50) SIGRTMAX-14	51) SIGRTMAX-13	52) SIGRTMAX-12
53) SIGRTMAX-11	54) SIGRTMAX-10	55) SIGRTMAX-9	56) SIGRTMAX-8	57) SIGRTMAX-7
58) SIGRTMAX-6	59) SIGRTMAX-5	60) SIGRTMAX-4	61) SIGRTMAX-3	62) SIGRTMAX-2
63) SIGRTMAX-1	64) SIGRTMAX
```
### 带外数据
带外数据用于迅速告知对方本端发生了重要的事件。它比普通的数据（带内数据）拥有更高的优先级，不论发送缓冲区中是否有排队等待发送的数据，它总是被立即发送。带外数据的传输可以使用一条独立的传输层连接，也可以映射到传输普通数据的连接中。实际应用中，带外数据是使用很少见，有telnet和ftp等远程非活跃程序。UDP没有没有实现带外数据传输，TCP也没有真正的带外数据。不过TCP利用头部的紧急指针标志和紧急指针，为应用程序提供了一种紧急方式，含义和带外数据类似。TCP的紧急方式利用传输普通数据的连接来传输紧急数据。

#### 内核如何通知应用程序带外数据?

内核通知应用程序带外数据到达的方式有两种:

* 利用IO复用技术的系统调用(select/poll/epoll)在接收到带外数据时将返回，并向应用程序报告socket上的异常时间。
* 使用SIGURG信号。


#### SIGURG
Go1.14新增了SIGURG信号，主要用于抢占P使用，主要解决for { //loop }这种场景。

具体请参见retake.md。