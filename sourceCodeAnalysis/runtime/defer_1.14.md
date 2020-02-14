# Go1.14 defer

Go 1.14版本对defer改动比较大，defer支持了open-coded模式。该模式可以对之前多次defer函数调用内联成一次，但该函数栈帧最动支持8个defer。优化之后defer性能提升比较明显，具体测试结果本篇不在过多赘述。


## open-coded模式
使用open-coded模式, panic处理非常不同。在生成函数代码时，编译器会生成额外的FUNCDATA信息集，该信息记录了每个open-coded defer信息。该FUNCDATA内指定了函数指针及每个参数的栈帧偏位置，它还生成了包含deferBits的stack槽位。由于stack可以伸缩，编译器对stack槽位使用varint编码。FUNCDATA全部使用varint编码,通过varp保存这些栈帧信息，具体格式如下:

- 所有defer中参数最大值
- deferBits变量偏移量
- 函数中defer数量
- 每个defer调用信息(该信息为调用顺序，和函数中defer注册顺序相反)
   
   * 调用的总参数大小
   * 要调用函数的偏移量(闭包函数)
   * 参数数量
   * 每一个参数具体信息
    
        * defer参数在栈帧中的偏移量
        * 参数的大小
        * 函数调用时，参数在args参数中的位置

## 源码剖析

#### _defer数据结构
Go1.14版本支持open-coded模式之后，_defer数据结构发生了改变。

```
// _defer是一个调用链,每注册一个_defer，相当于向该调用链压入一个_defer.
type _defer struct {
	siz     int32
	started bool    // 当前_defer是否已经被执行
	heap    bool 	// 发生逃逸,_defer分配在堆上

	
	// 当有多个defer需要激活时，使用openDefer模式，可以优化为只有一个defer。
	// 如果openDefer开启,使用open-coded最大defer限制为8,因为deferBits当前是一个字节.
	// ssa.go: const maxOpenDefers = 8
	// defer 类型: open-coded, stack-allocated, heap-allocated
	// 更多信息请参考: https://github.com/golang/proposal/blob/master/design/34481-opencoded-defers.md
	openDefer bool	   // 是否开启openDefer模式
	sp        uintptr  // sp at time of defer
	pc        uintptr  // pc at time of defer
	fn        *funcval // open-coded该fn为nil
	_panic    *_panic  // panic在defer之后触发
	link      *_defer

	
	// openDefer开启之后,以下字段才会被使用。
	// 这些字段会记录open-coded defer stack和相关的函数值

    // 所有FUNCDATA数据保存在该栈帧
	fd unsafe.Pointer // fd => funcdata

	// varp指向的栈帧
	varp uintptr        // value of varp for the stack frame
	framepc uintptr     // pc/sp ==> framepc/sp
}
```

### deferprocStack
defer注册时，会调用deferproc或deferproceStack，具体改动不大。

```
// deferprocStack 将_defer压入栈上。等待函数退出时, 从stack拿出defer函数调用
func deferprocStack(d *_defer) {
	gp := getg()

    // defer初始化...
	d.started = false
	d.heap = false
	d.openDefer = false        	// open-coded模式，1.14版本新增
	d.sp = getcallersp()
	d.pc = getcallerpc()
	d.framepc = 0			 	// open-coded模式，1.14版本新增
	d.varp = 0					// open-coded模式，保存funcdata内容。 1.14版本新增


	// The lines below implement:
	//   d.panic = nil
	//   d.fd = nil
	//   d.link = gp._defer
	//   gp._defer = d
	*(*uintptr)(unsafe.Pointer(&d._panic)) = 0
	*(*uintptr)(unsafe.Pointer(&d.fd)) = 0
	*(*uintptr)(unsafe.Pointer(&d.link)) = uintptr(unsafe.Pointer(gp._defer))     // 未初始化内存
	
    // 将创建defer 压栈
    *(*uintptr)(unsafe.Pointer(&gp._defer)) = uintptr(unsafe.Pointer(d))

	return0()
	// No code can go here - the C return register has
	// been set and must not be clobbered.
}
```

### deferreturn
当函数触发return函数之后，会调用deferreturn函数，依次会执行defer链表内函数。这也是Go1.14版本defer改动最大地方，使用defer-coded将多次defer调用内联成一次。极大提升defer性能。

```
func deferreturn(arg0 uintptr) {
	gp := getg()

	// 当前G栈顶_defer,即最后一个注册的_defer
	d := gp._defer
	sp := getcallersp()

	// 是否启用open-coded模式,1.14新功能
	// 函数inline,提升性能
	if d.openDefer {
		done := runOpenDeferFrame(gp, d)
		if !done {
			throw("unfinished open-coded defers in deferreturn")
		}

		// defer stack释放
		gp._defer = d.link
		freedefer(d)
		return
	}


	// 之前版本相同,具体解析查看defer.md
	switch d.siz {
	case 0:
		// Do nothing.
	case sys.PtrSize:
		*(*uintptr)(unsafe.Pointer(&arg0)) = *(*uintptr)(deferArgs(d))
	default:
		memmove(unsafe.Pointer(&arg0), deferArgs(d), uintptr(d.siz))
	}
	fn := d.fn
	d.fn = nil
	gp._defer = d.link
	freedefer(d)


	jmpdefer(fn, uintptr(unsafe.Pointer(&arg0)))
}
```

#### runOpenDeferFrame
deferreturn函数defer-coded具体实现都在该函数。

```
// 一个stack空间执行所有defer函数, 1.14版本defer性能优化
func runOpenDeferFrame(gp *g, d *_defer) bool {
	done := true

	// 所有defer信息都存储在funcdata中
	fd := d.fd

	_, fd = readvarintUnsafe(fd)                  	// 跳过maxargssize
	deferBitsOffset, fd := readvarintUnsafe(fd)		// deferBits偏移量
 	nDefers, fd := readvarintUnsafe(fd)				// 函数中defer数量
	deferBits := *(*uint8)(unsafe.Pointer(d.varp - uintptr(deferBitsOffset)))   // 当前执行的defer

	for i := int(nDefers) - 1; i >= 0; i-- {
		// 从defer中读取funcdata信息
		var argWidth, closureOffset, nArgs uint32
		argWidth, fd = readvarintUnsafe(fd)
		closureOffset, fd = readvarintUnsafe(fd)
		nArgs, fd = readvarintUnsafe(fd)

		// 执行的函数和参数
		closure := *(**funcval)(unsafe.Pointer(d.varp - uintptr(closureOffset)))
		d.fn = closure
		deferArgs := deferArgs(d)

		// 将_defer参数数据拷贝到和_defer关联内存位置
		for j := uint32(0); j < nArgs; j++ {
			//      目标       <=            源
			// argCallOffset  <=    argOffset + argLen
			var argOffset, argLen, argCallOffset uint32		// defer参数在栈帧中的偏移量
			argOffset, fd = readvarintUnsafe(fd)			// 参数大小
			argLen, fd = readvarintUnsafe(fd)				// 函数调用时，参数在args参数中的位置
			argCallOffset, fd = readvarintUnsafe(fd)
			memmove(unsafe.Pointer(uintptr(deferArgs)+uintptr(argCallOffset)),
				unsafe.Pointer(d.varp-uintptr(argOffset)),
				uintptr(argLen))
		}

		// 通过位移操作,移到下一个defer执行位置
		deferBits = deferBits &^ (1 << i)
		*(*uint8)(unsafe.Pointer(d.varp - uintptr(deferBitsOffset))) = deferBits

		// 通过reflect方式执行defer fn函数
		// reflect有性能消耗!!! 
		p := d._panic
		reflectcallSave(p, unsafe.Pointer(closure), deferArgs, argWidth)
		if p != nil && p.aborted {
			break
		}
		d.fn = nil
		
		// 清理defer参数内存空间，defer只是一个复制品
		memclrNoHeapPointers(deferArgs, uintptr(argWidth))
		if d._panic != nil && d._panic.recovered {
			done = deferBits == 0
			break
		}
	}

	return done
}
```

## 总结
defer改动牵扯panic、Goexit也发生比较大改变。具体请参考panic 1.14性能优化。