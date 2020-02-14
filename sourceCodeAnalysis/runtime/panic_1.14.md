# panic
Go 1.14版本对defer进行了性能优化，随之panic自然也需要进行改动。panic 改动也比较大。具体改动就是gopanic函数，本篇主要剖析gopanic改动。

## 源码剖析
panic异常触发主要由gopanic函数实现，具体剖析如下。

### gopanic

```
func gopanic(e interface{}) {
	gp := getg()

	// 创建_panic,挂到G._panic链表
	var p _panic
	p.arg = e
	p.link = gp._panic
	gp._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

	atomic.Xadd(&runningPanicDefers, 1)

	// 1. 扫描gopanic栈帧的defer，将找的的defer依次押入到defer链
	// 2. 然后新建一个defer,压入到defer链,用于关联当前的gopanic
	addOneOpenDeferFrame(gp, getcallerpc(), unsafe.Pointer(getcallersp()))

	for {
		d := gp._defer

		// 如果当前g中的defer已经启动(defer在panic之前已经被创建),则将当前defer从defer链上删除
		if d.started {
			// 之前的panic不会被执行,标记为aborted
			if d._panic != nil {
				d._panic.aborted = true
			}

			d._panic = nil
			if !d.openDefer {
				// 将当前defer从defer链中删除
				d.fn = nil
				gp._defer = d.link
				freedefer(d)
				continue
			}
		}

		// 当前defer打上开始标签
		d.started = true
 
		// 当前defer和panic进行关联
		// 如果在此期间出现新的Panic,则该panic重新在defer链找一个defer进行关联,但d._panic.aborted标记为true.
		d._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

		done := true

		// 开启了open-coded 模式, open-coded模式只需要一个defer即可完成之前多个defer的任务，
		// 无需多次defer调用。 提升性能!!!
		if d.openDefer {
			// 通过open-coded模式执行defer fn函数
			done = runOpenDeferFrame(gp, d)

			// 如果未开启recover,则循环继续执行defer链其它defer fn
			if done && !d._panic.recovered {
				addOneOpenDeferFrame(gp, 0, nil)
			}
		} else {

			// 普通模式，执行defer fn函数
			p.argp = unsafe.Pointer(getargp(0))
			reflectcall(nil, unsafe.Pointer(d.fn), deferArgs(d), uint32(d.siz), uint32(d.siz))
		}

		// 释放defer
		p.argp = nil
		d._panic = nil
		pc := d.pc
		sp := unsafe.Pointer(d.sp) // must be pointer so it gets adjusted during stack copy
		if done {
			d.fn = nil
			gp._defer = d.link
			freedefer(d)
		}

		// 调用过recover函数
		if p.recovered {
			gp._panic = p.link
			if gp._panic != nil && gp._panic.goexit && gp._panic.aborted {
				// A normal recover would bypass/abort the Goexit.  Instead,
				// we return to the processing loop of the Goexit.
				gp.sigcode0 = uintptr(gp._panic.sp)
				gp.sigcode1 = uintptr(gp._panic.pc)
				mcall(recovery)
				throw("bypassed recovery failed") // mcall should not return
			}

			if done {
				// 将defer链内所有defer删除
				d := gp._defer
				var prev *_defer
				for d != nil {
					if d.openDefer {
						if prev == nil {
							gp._defer = d.link
						} else {
							prev.link = d.link
						}
						freedefer(d)
						break
					} else {
						prev = d
						d = d.link
					}
				}
			}

			// 此时,如果panics链表内还有panic,则将该列表的panic删除
			gp._panic = p.link
			for gp._panic != nil && gp._panic.aborted {
				gp._panic = gp._panic.link
			}

			// 标记完成信号
			if gp._panic == nil {
				gp.sig = 0
			}


			// 恢复栈帧
			gp.sigcode0 = uintptr(sp)
			gp.sigcode1 = pc
			mcall(recovery)
			throw("recovery failed")
		}
	}
}
```

### addOneOpenDeferFrame
addOneOpenDeferFrame主要扫描当前栈帧所有defer，依次使用defer-coded模式压入defer链。最后创建一个新的defer和panic进行关联。

```
// addOneOpenDeferFrame 扫描整个gopanic栈帧的defer，将找的的defer押入defer链；扫描完成,新增一个open-coded defer,加入到defer链,退出扫描
// 
// 注意: 如果defer运行期间,stack发生改变，则所有的defer记录都需要改变.同样，sp可以作为参数进行传递,因为stack复制期间，需要调整所有指针变量.
func addOneOpenDeferFrame(gp *g, pc uintptr, sp unsafe.Pointer) {
	var prevDefer *_defer

	systemstack(func() {
		gentraceback(pc, uintptr(sp), 0, gp, 0, nil, 0x7fffffff,
			func(frame *stkframe, unused unsafe.Pointer) bool {
				if prevDefer != nil && prevDefer.sp == frame.sp {
					// 跳过上一个defer栈帧，从当前位置继续扫描下一个defer栈帧
					return true
				}
				f := frame.fn
				fd := funcdata(f, _FUNCDATA_OpenCodedDeferInfo)
				if fd == nil {
					return true
				}

				d := gp._defer
				var prev *_defer
				for d != nil {
					dsp := d.sp
					if frame.sp < dsp {
						break
					}
					if frame.sp == dsp {
						return true
					}

					// 扫描到的defer插入到defer链,继续扫描
					prev = d
					d = d.link
				}


				// 扫描完之后，创建一个open-coded defer,然后退出扫描
				maxargsize, _ := readvarintUnsafe(fd)
				d1 := newdefer(int32(maxargsize))
				d1.openDefer = true
				d1._panic = nil
				d1.pc = frame.fn.entry + uintptr(frame.fn.deferreturn)
				d1.varp = frame.varp
				d1.fd = fd
				d1.framepc = frame.pc
				d1.sp = frame.sp
				
				// 新创建defer压入defer链
				d1.link = d
				if prev == nil {
					gp._defer = d1
				} else {
					prev.link = d1
				}

				// 添加一个open-coded defer之后，停止扫描
				return false
			},
			nil, 0)
	})
}
```

## 总结

结束