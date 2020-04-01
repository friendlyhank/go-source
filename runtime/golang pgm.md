 - P用来**控制并发执行任务数量**,每个工作线程都要绑定一个有效的P才能被执行,它的数量大于等于CPU数量。
 - Goroutine(简称G)是Go最小的执行单元,事实上每个go程序至少会有一个Goroutine：主Goroutine,当程序启动时就会自动创建。//TODO groutine是任务要再补充说明
 - M是系统线程抽象(简称M)，它和P绑定,以调度循环方式不停执行G任务。M通过修改寄存器，将执行栈指向G自带栈内存，并在此空间内分配堆栈帧，执行任务函数。当需要中途切换时，只要将相关寄存器值保存回G空间即可维持状态,任何M都可据此恢复执行。线程仅负责执行，不再持有状态，这是并发任务跨线程调度，实现多路复用的根本所在。

因为G初始栈仅有2KB

M在绑定了有效的P后，调度器schedule让多个M进入循环，不停获取并执行任务，所以我们才能创建上万个并发任务。

一下是在go1.41版本：

runtime/runtime2.go
调度器的结构体定义：
```go
type schedt struct {
	//M相关
	midle        muintptr //空闲的m链表(复用)
	nmidle       int32//空闲m数量
	maxmcount    int32 //定义M的最大数量
	//P相关
	pidle      puintptr // 空闲的p链表
	npidle     uint32 //空闲的数量
	
	runq     gQueue  //全局的队列
	runqsize int32  //队列的大小
}
```

M的结构体定义：
```go
type m struct {
	g0      *g  //调度栈的goroutine
	curg          *g //当前线程正在运行的Goroutine
}
```
g0是持有调度栈的Goroutine,curg是当前线程正在运行的Goroutine,这也是操作系统线程唯一关心的两个Goroutine。
g0 是一个运行时中比较特殊的 Goroutine，它会深度参与运行时的调度过程，包括 Goroutine 的创建、大内存分配和 CGO 函数的执行。

P的结构体定义：
```go
type p struct {
	runqhead uint32   //头队列
	runqtail uint32   //尾队列
	runq     [256]guintptr //本地队列
	runnext guintptr//优先队列
	//空闲g复用
	gFree struct {
		gList
		n int32
	}
}
```

G的结构体定义：
```go
type g struct {
	stack       stack //执行栈
	goid         int64 //唯一序号
}
```

## 调度器的初始化
在执行main函数的时候调度器会完成初始化,初始化文件在runtime/proc.go:
```go
func schedinit() {
     //设置M的最大值
	sched.maxmcount = 10000
	
	//初始化当前m
	mcommoninit(_g_.m)
	
	//初始化P的数量
	if procresize(procs) != nil {
		throw("unknown runnable goroutine during bootstrap")
	}
}
```

方法总结：
```go
procresize
```

## 创建Goroutine
在main方法或者是具有go func()()关键字的时候,编译器会调用newproc创建Goroutine,具体文件路径runtime/proc.go
```go
func newproc(siz int32, fn *funcval) {
	argp := add(unsafe.Pointer(&fn), sys.PtrSize)
	gp := getg()
	pc := getcallerpc()
	systemstack(func() {
		newproc1(fn, argp, siz, gp, pc)
	})
}
```

### 创建步骤
- 创建新的Goroutine
- 将传入的参数压入栈中
- 更新Goroutine调度相关的属性
- 将Goroutine加入处理器的运行队列

**创建新的Gorountine**
```powershell
func newproc1(fn *funcval, argp unsafe.Pointer, narg int32, callergp *g, callerpc uintptr) {
	_p_ := _g_.m.p.ptr()
	newg := gfget(_p_)
	if newg == nil {
		newg = malg(_StackMin)
		casgstatus(newg, _Gidle, _Gdead)
		allgadd(newg) // publishes with a g->status of Gdead so GC scanner doesn't look at uninitialized stack.
	}
}
```
创建goroutine过程通过gfget函数,会先从处理器p的gFree列表中查找空闲的Gorountine,如果不存在空闲的Gorountine会从调度器schedt的gFree中取,如果还是不存在就会调用malg创建新的g。

接下来会调用memmove函数将fn函数的所有参数压入栈中

```powershell
...
if narg > 0 {
	memmove(unsafe.Pointer(spArg), argp, uintptr(narg))
}
...
```

拷贝了栈上参数之后,runtime.newproc1会设置新的Gorountine结构体的参数，包括栈指针、程序计数器并更新其状态到 **_Grunnable**

```powershell
...
memclrNoHeapPointers(unsafe.Pointer(&newg.sched), unsafe.Sizeof(newg.sched))
	newg.sched.sp = sp
	newg.stktopsp = sp
	newg.sched.pc = funcPC(goexit) + sys.PCQuantum // +PCQuantum so that previous instruction is in same function
	newg.sched.g = guintptr(unsafe.Pointer(newg))
	gostartcallfn(&newg.sched, fn)
	newg.gopc = callerpc
	newg.ancestors = saveAncestors(callergp)
	newg.startpc = fn.fn
	if _g_.m.curg != nil {
		newg.labels = _g_.m.curg.labels
	}
	if isSystemGoroutine(newg, false) {
		atomic.Xadd(&sched.ngsys, +1)
	}
	//更新g的状态
	casgstatus(newg, _Gdead, _Grunnable)

	if _p_.goidcache == _p_.goidcacheend {
		// Sched.goidgen is the last allocated id,
		// this batch must be [sched.goidgen+1, sched.goidgen+GoidCacheBatch].
		// At startup sched.goidgen=0, so main goroutine receives goid=1.
		_p_.goidcache = atomic.Xadd64(&sched.goidgen, _GoidCacheBatch)
		_p_.goidcache -= _GoidCacheBatch - 1
		_p_.goidcacheend = _p_.goidcache + _GoidCacheBatch
	}
	newg.goid = int64(_p_.goidcache)
	_p_.goidcache++
...
```
在最后,runqput会将初始化好的Goroutine放入队列之中,然后唤起线程wakep取执行。

## 运行队列
runtime.runqput将会在新建的Goroutine运行的队列上,这既可能会在全局的运行队列,也可能处理器的本地队列。

runtime/proc.go
```powershell
func runqput(_p_ *p, gp *g, next bool) {
	if randomizeScheduler && next && fastrand()%2 == 0 {
		next = false
	}

	if next {
	retryNext:
		oldnext := _p_.runnext
		if !_p_.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {
			goto retryNext
		}
		if oldnext == 0 {
			return
		}
		// Kick the old runnext out to the regular run queue.
		gp = oldnext.ptr()
	}

retry:
	h := atomic.LoadAcq(&_p_.runqhead) // load-acquire, synchronize with consumers
	t := _p_.runqtail
	if t-h < uint32(len(_p_.runq)) {
		_p_.runq[t%uint32(len(_p_.runq))].set(gp)
		atomic.StoreRel(&_p_.runqtail, t+1) // store-release, makes the item available for consumption
		return
	}
	//全局要加锁，所以slow
	if runqputslow(_p_, gp, h, t) {
		return
	}
	// the queue is not full, now the put above must succeed
	goto retry
}
```

 1. 当 next 为 true 时，将 Goroutine 设置到处理器的 runnext 上作为下一个处理器执行的任务。
 2. 当 next 为 false 并且本地运行队列还有剩余空间时，将 Goroutine 加入处理器持有的本地运行队列；
 3. 当处理器的本地运行队列已经满了就会把本地队列中的一部分 Goroutine 和待加入的 Goroutine 通过 runqputslow 添加到调度器持有的全局运行队列上,新建的Goroutine也会被放到全局运行队列中；

可以看出任务队列分为三级，按优先级从高到低分别是P.runnext、P.runq、sched.runq。
P.runnext、P.runq是无锁操作,sched.runq全局有锁操作。

**Goroutine执行完毕,调度器相关函数G对象会放回P复用链表**
```powershell
// Put on gfree list.
// If local list is too long, transfer a batch to the global list.
func gfput(_p_ *p, gp *g) {
	if readgstatus(gp) != _Gdead {
		throw("gfput: bad status (not Gdead)")
	}

	stksize := gp.stack.hi - gp.stack.lo

	if stksize != _FixedStack {
		// non-standard stack size - free it.
		stackfree(gp.stack)
		gp.stack.lo = 0
		gp.stack.hi = 0
		gp.stackguard0 = 0
	}
	
	//放回P本地复用链表
	_p_.gFree.push(gp)
	_p_.gFree.n++
	//如果本地(p)复用链表过多，则转移一批到全局列表
	if _p_.gFree.n >= 64 {
		lock(&sched.gFree.lock)
		for _p_.gFree.n >= 32 {
			_p_.gFree.n--
			gp = _p_.gFree.pop()
			if gp.stack.lo == 0 {
				sched.gFree.noStack.push(gp)
			} else {
				sched.gFree.stack.push(gp)
			}
			sched.gFree.n++
		}
		unlock(&sched.gFree.lock)
	}
}
```


## 执行
```go
func mstart() {
	mstart1()
}

func mstart1() {
	if fn := _g_.m.mstartfn; fn != nil {
		fn()
	}
	//进入任务调度循环
	schedule()
}
```

```go
func schedule() {
	//每处理n个任务后就去全局队列获取G任务，以确保公平
	if gp == nil {
		if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
			lock(&sched.lock)
			gp = globrunqget(_g_.m.p.ptr(), 1)
			unlock(&sched.lock)
		}
	}
	//优先获取本地的优先队列，再获取本地队列
	if gp == nil {
		gp, inheritTime = runqget(_g_.m.p.ptr())
		// We can see gp != nil here even if the M is spinning,
		// if checkTimers added a local goroutine via goready.
	}
	if gp == nil {
		gp, inheritTime = findrunnable() // blocks until work is available
	}
	//执行任务函数
	execute(gp, inheritTime)
}
```

```go
func findrunnable() (gp *g, inheritTime bool) {
	// 再次从本地队列获取
	if gp, inheritTime := runqget(_p_); gp != nil {
		return gp, inheritTime
	}
	
	//从全局队列获取
	if sched.runqsize != 0 {
		lock(&sched.lock)
		gp := globrunqget(_p_, 0)
		unlock(&sched.lock)
		if gp != nil {
			return gp, false
		}
	}
	
	for i := 0; i < 4; i++ {
		//随机挑一个p偷些任务
		for enum := stealOrder.start(fastrand()); !enum.done(); enum.next() {
			if sched.gcwaiting != 0 {
				goto top
			}
			stealRunNextG := i > 2 // first look for ready queues with more than 1 g
			p2 := allp[enum.position()]
			if _p_ == p2 {
				continue
			}
			if gp := runqsteal(_p_, p2, stealRunNextG); gp != nil {
				return gp, false
			}
		}
	}
}
```
为了保证公平，当全局运行队列中有待执行的 Goroutine 时，通过 schedtick 保证有一定几率会从全局的运行队列中查找对应的 Goroutine；
从处理器本地的运行队列中查找待执行的 Goroutine；
如果前两种方法都没有找到 Goroutine，就会通过 runtime.findrunnable 进行阻塞地查找 Goroutine

从本地运行队列、全局运行队列中查找；
从网络轮询器中查找是否有 Goroutine 等待运行；
通过 runtime.runqsteal 函数尝试从其他随机的处理器中窃取待运行的 Goroutine，在该过程中还可能窃取处理器中的计时器；

参考:
[https://speakerdeck.com/retervision/go-runtime-scheduler](https://speakerdeck.com/retervision/go-runtime-scheduler)
[https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#heading-10](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#heading-10)




