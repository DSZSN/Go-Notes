# forceGC

forceGC用于检查在2分钟内是否触发过GC，如果未触发，则由sysmon强制执行GC。

## 源码剖析

###  forceGC 使用全局变量

```
// runtime/runtime2.go
var forcegc    forcegcstate

// 强制触发GC时间， 2min
var forcegcperiod int64 = 2 * 60 * 1e9
```

### gcTrigger.test

```
// 测试是否满足强制触发GC条件
func (t gcTrigger) test() bool {
	switch t.kind {
	case gcTriggerHeap:
		return memstats.heap_live >= memstats.gc_trigger
	case gcTriggerTime:
		// 如果超过2min没有触发GC，则强制执行GC
		lastgc := int64(atomic.Load64(&memstats.last_gc_nanotime))
		return lastgc != 0 && t.now-lastgc > forcegcperiod
	case gcTriggerCycle:
		return int32(t.n-work.cycles) > 0
	}
	return true
}
```

```
type forcegcstate struct {
	lock mutex
	g    *g
	idle uint32
}
```