# Go Directives

编程中,编译指令(pragma)是一种语言结构，它指示编译器应该如何处理输入。指令不是编程语言语法的一部分，因编译器而异。

> [Wikipad](https://en.wikipedia.org/wiki/Directive_%28programming%29)


## Go 语言的编译指令

格式//go:就是Go语言编译指令的实现方式。如下是Go语言支持的全部编译指令。

```
func pragmaValue(verb string) syntax.Pragma {
	switch verb {
	case "go:nointerface":
		if objabi.Fieldtrack_enabled != 0 {
			return Nointerface
		}
	case "go:noescape":
		return Noescape
	case "go:norace":
		return Norace
	case "go:nosplit":
		return Nosplit | NoCheckPtr // implies NoCheckPtr (see #34972)
	case "go:noinline":
		return Noinline
	case "go:nocheckptr":
		return NoCheckPtr
	case "go:systemstack":
		return Systemstack
	case "go:nowritebarrier":
		return Nowritebarrier
	case "go:nowritebarrierrec":
		return Nowritebarrierrec | Nowritebarrier // implies Nowritebarrier
	case "go:yeswritebarrierrec":
		return Yeswritebarrierrec
	case "go:cgo_unsafe_args":
		return CgoUnsafeArgs | NoCheckPtr // implies NoCheckPtr (see #34968)
	case "go:uintptrescapes":
		// For the next function declared in the file
		// any uintptr arguments may be pointer values
		// converted to uintptr. This directive
		// ensures that the referenced allocated
		// object, if any, is retained and not moved
		// until the call completes, even though from
		// the types alone it would appear that the
		// object is no longer needed during the
		// call. The conversion to uintptr must appear
		// in the argument list.
		// Used in syscall/dll_windows.go.
		return UintptrEscapes
	case "go:notinheap":
		return NotInHeap
	}
	return 0
}
```

> [官方文档](https://golang.org/cmd/compile/#hdr-Compiler_Directives)

### 常用指令

## noinline

go:noinline不需要内联。

##### Inline 内联
> Inline,是在编译期间发生的，将函数调用处替换为被调用函数主体的一种编译器优化手段。

##### Inline优缺点

* 优势
    
    * 减少函数调用的开销，提高执行速度。
    * 复制后的更大函数体为其他编译优化带来可能性。
    * 消除分支，并改善空间局部性和指令顺序性，同样可以提高性能。

* 问题

    * 代码复制带来的空间增长。
    * 如果有大量重复代码，反而会降低缓存命中率，尤其对 CPU 缓存是致命的。

所以，在实际使用中，对于是否使用内联，需要谨慎考虑，并做好平衡，以使它发挥最大的作用。简单来说，对于短小而且工作较少的函数，使用内联是有效益的。

##### 例子

```
package main

func H() int {
        return 0x64
}

func main() {
	d := H()
    println(d)
}
```

* 内联
```
shell> go build -o test main.go
shell> go tool objdump -s "main\.main" test
TEXT main.main(SB) /Users/shszzhang/data/mine/go.test/src/example/test.go
  test.go:8		0x1051570		65488b0c2530000000	MOVQ GS:0x30, CX
  test.go:8		0x1051579		483b6110		CMPQ 0x10(CX), SP
  test.go:8		0x105157d		7634			JBE 0x10515b3
  test.go:8		0x105157f		4883ec10		SUBQ $0x10, SP
  test.go:8		0x1051583		48896c2408		MOVQ BP, 0x8(SP)
  test.go:8		0x1051588		488d6c2408		LEAQ 0x8(SP), BP
  test.go:10		0x105158d		e82e3afdff		CALL runtime.printlock(SB)
  test.go:10		0x1051592		48c7042464000000	MOVQ $0x64, 0(SP)           // 内联
  test.go:10		0x105159a		e8a141fdff		CALL runtime.printint(SB)
  test.go:10		0x105159f		e8ac3cfdff		CALL runtime.printnl(SB)
  test.go:10		0x10515a4		e8973afdff		CALL runtime.printunlock(SB)
  test.go:11		0x10515a9		488b6c2408		MOVQ 0x8(SP), BP
  test.go:11		0x10515ae		4883c410		ADDQ $0x10, SP
  test.go:11		0x10515b2		c3			RET
  test.go:8		0x10515b3		e84881ffff		CALL runtime.morestack_noctxt(SB)
  test.go:8		0x10515b8		ebb6			JMP main.main(SB)
```

* 禁止内联

```
//go:noinline
func H() int {
        return 0x64
}

shell> go build -o test main.go
shell> go tool objdump -s "main\.main" test

TEXT main.main(SB) /Users/shszzhang/data/mine/go.test/src/example/test.go
  test.go:8		0x1051580		65488b0c2530000000	MOVQ GS:0x30, CX
  test.go:8		0x1051589		483b6110		CMPQ 0x10(CX), SP
  test.go:8		0x105158d		7643			JBE 0x10515d2
  test.go:8		0x105158f		4883ec18		SUBQ $0x18, SP
  test.go:8		0x1051593		48896c2410		MOVQ BP, 0x10(SP)
  test.go:8		0x1051598		488d6c2410		LEAQ 0x10(SP), BP
  test.go:9		0x105159d		e8ceffffff		CALL main.H(SB)                        // 禁止内联,直接函数调用
  test.go:9		0x10515a2		488b0424		MOVQ 0(SP), AX
  test.go:9		0x10515a6		4889442408		MOVQ AX, 0x8(SP)
  test.go:10		0x10515ab		e8103afdff		CALL runtime.printlock(SB)
  test.go:10		0x10515b0		488b442408		MOVQ 0x8(SP), AX
  test.go:10		0x10515b5		48890424		MOVQ AX, 0(SP)
  test.go:10		0x10515b9		e88241fdff		CALL runtime.printint(SB)
  test.go:10		0x10515be		e88d3cfdff		CALL runtime.printnl(SB)
  test.go:10		0x10515c3		e8783afdff		CALL runtime.printunlock(SB)
  test.go:11		0x10515c8		488b6c2410		MOVQ 0x10(SP), BP
  test.go:11		0x10515cd		4883c418		ADDQ $0x18, SP
  test.go:11		0x10515d1		c3			RET
  test.go:8		0x10515d2		e82981ffff		CALL runtime.morestack_noctxt(SB)
  test.go:8		0x10515d7		eba7			JMP main.main(SB)
```

## nosplit
nosplit 作用是跳过stack溢出检测。

##### stack溢出？
Go语言初始化stack大小是2KB，相比系统栈小很多，才可以做到支持高并发Goroutine，实现高效调度。Goroutine栈大小是动态变化的,当不够用时，它会动态增长。nosplit将跳过该机制，不对该stack增长。

##### 优缺点
不执行stack溢出检查，可以提高性能，但同时也有可能发生stack overflow而导致编译失败。


## noescape
noescape作用是禁止逃逸，从stack逃逸到heap上。

##### 逃逸是什么？
Go相比C、C++是内存更加安全的语言。它可以自动将超出自身生命周期的变量，从stack逃逸到heap上，逃逸指这种行为。逃逸之后，性能会降低(内存需要GC进行回收，触发STW)。

##### 优缺点

* 优点

    noescape可以提升性能，stack通过sp寄存器自动内存伸缩。

* 缺点

    noescape禁止逃逸后，当函数返回结果为共享资源时(其它函数也需要该资源),会导致异常。


## norace
norace禁止竞争检查。在并发编程中，难免会出现数据竞争，正常情况下，当编译器检测到有数据竞争时,就会给出提示。

```
package main

var sum int

func main() {
	go add()
	go add()
}

func add() {
	sum++
}
```

##### 数据竞争检查

```
shell> go run -race race.go
==================
WARNING: DATA RACE
Read at 0x000001137698 by goroutine 7:
  main.add()
      ~/data/mine/go.test/src/example/race.go:11 +0x3a

Previous write at 0x000001137698 by goroutine 6:
  main.add()
      ~/data/mine/go.test/src/example/race.go:11 +0x56

Goroutine 7 (running) created at:
  main.main()
      ~/data/mine/go.test/src/example/race.go:7 +0x5a

Goroutine 6 (finished) created at:
  main.main()
      ~/data/mine/go.test/src/example/race.go:6 +0x42
==================
Found 1 data race(s)
exit status 66
```

##### 关闭数据竞争检查

```
//go:norace
func add() {
	sum++
}
```

##### 优缺点

使用norace可以缩短编译时间。缺点就是并发编程中，如果出现数据竞争，会导致程序运行不确定性。


## 总结

Go还有很多其它编译器指令，可以根据需要进行选择使用。