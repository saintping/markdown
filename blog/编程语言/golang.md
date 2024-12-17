### 前言
第一次接触golang是16年左右，在腾讯云做COS服务时期。后来随着K8S和虚拟币的持续火爆，golang也最终进入大部分程序员的视野。

golang是Ken Thompson（Unix&C的主程）在google工作时写的，前身是其编写的实验性语言C-。golang的编译器最开始是C语言写的，后来用golang重写了。和C++以繁杂晦涩著称的学术性规范不同，golang是一门非常工程化的语言。
[https://go.dev/ref/spec](https://go.dev/ref/spec "https://go.dev/ref/spec")

可以粗暴的理解为`golang = struct + interface + coroutine + gc`。这里简单对比一下golang和C以及其他高级语言的差异。


### 常见语法
所有关键字只有25个，这在高级语言里是很克制的。对比c语言有30+个，java有50+，C++有70+。
```go
break        default      func         interface    select
case         defer        go           map          struct
chan         else         goto         package      switch
const        fallthrough  if           range        type
continue     for          import       return       var
```

- 类型后置
  类型后置应该是劝退新手的第一大因素，感受一下。
  ```go
  //变量
  var x interface{}  // x is nil and has static type interface{}
  var v *T           // v has value nil, static type *T
  x = 42             // x has value 42 and dynamic type int
  x = v              // x has value (*T)(nil) and dynamic type *T

  //切片
  v = make([]int, 50, 100)
  x = v[1, 10]

  //结构
  struct {
      x, y int
      u float32
      _ float32  // padding
      A *[]int
      F func()
  }

  //通道
  x = chan T          // can be used to send and receive values of type T
  x = chan<- float64  // can only be used to send float64s
  x = <-chan int      // can only be used to receive ints

  //函数
  func foo(x int) int
  func foo(a, b int, z float32) (bool)
  func foo(int, int, float64) (float64, *[]int)
  func foo(n int) func(p *T)

  //方法
  func (p T) Read(p []byte) (n int, err error)
  func (p T) Write(p []byte) (n int, err error)
  func (p T) Close() error

  //接口
  type Reader interface {
      Read(p []byte) (n int, err error)
      Close() error
  }
  type Writer interface {
      Write(p []byte) (n int, err error)
      Close() error
  }
  // ReadWriter's methods are Read, Write, and Close.
  type ReadWriter interface {
      Reader  // includes methods of Reader in ReadWriter's method set
      Writer  // includes methods of Writer in ReadWriter's method set
  }
  ```

- 更简单灵活循环for
  没有while关键字，全部靠for
  
- 函数与闭包closure
  函数和闭包运行时使用同一个结构funcval。闭包就是带有自由变量的函数。
  ```go
  type funcval struct {
      fn uintptr
      // variable-size, fn-specific data here
  }
  ```

- 接口interface
  接口是作用于struct的函数集合。有点类似C++里的vtable，只是golang把他从对象上完全独立出来了。对象可以随时定义新的接口。运行时结构如下：
  ```go
  type ITab struct {
      Inter *InterfaceType
      Type  *Type
      Hash  uint32     // copy of Type.Hash. Used for type switches.
      Fun   [1]uintptr // variable sized. fun[0]==0 means Type does not implement Inter.
  }
  type itab = abi.ITab
  ```


### 运行时语义
这内容大部分都在这个文件里`src/runtime/runtime2.go`

- 协程coroutine
  大部分调度设计都有三个角色：调度P，执行者M和例程G，golang也不例外。比如Linux内核中：执行者是cpu，调度是schedule，例程是task；golang和Java的协程结构非常类似：执行者是系统线程，调度是schedule，例程是coroutine；但是golang开启协程非常简单，在任何函数调用前go即可。
  go关键字在运行期对应的是newproc，创建一个g放到一个全局队列中等待调度。每个执行者有他自己本地Steal模式的队列。
  ```go
  // Create a new g running fn.
  // Put it on the queue of g's waiting to run.
  // The compiler turns a go statement into a call to this.
  func newproc(fn *funcval) {
  	gp := getg()
  	pc := sys.GetCallerPC()
  	systemstack(func() {
  		newg := newproc1(fn, gp, pc, false, waitReasonZero)
  
  		pp := getg().m.p.ptr()
  		runqput(pp, newg, true)
  
  		if mainStarted {
  			wakep()
  		}
  	})
  }
  ```
  ![https://ping666.com/wp-content/uploads/2024/12/goroutine.webp](https://ping666.com/wp-content/uploads/2024/12/goroutine.webp "goroutine.webp")

- 通道channel
  如果是一个channel，可以认为是一个FIFO队列的语法糖。但是多个channel配合select，支持类似多路复用的逻辑，威力就大了。
  ```go
  var a []int
  var c, c1, c2, c3, c4 chan int
  var i1, i2 int
  select {
  case i1 = <-c1:
  	print("received ", i1, " from c1\n")
  case c2 <- i2:
  	print("sent ", i2, " to c2\n")
  case i3, ok := (<-c3):  // same as: i3, ok := <-c3
  	if ok {
  		print("received ", i3, " from c3\n")
  	} else {
  		print("c3 is closed\n")
  	}
  case a[f()] = <-c4:
  	// same as:
  	// case t := <-c4
  	//	a[f()] = t
  default:
  	print("no communication\n")
  }
  ```
- 延迟调用defer
  在函数执行完成，return之前（严谨一点来说是ret指令）执行的函数。有点类似Java里finally的味道。可以定义多个构成一个列表，后进先出。
  ```go
  type _defer struct {
  	heap      bool
  	rangefunc bool    // true for rangefunc list
  	sp        uintptr // sp at time of defer
  	pc        uintptr // pc at time of defer
  	fn        func()  // can be nil for open-coded defers
  	link      *_defer // next defer on G; can point to either heap or stack!
  
  	// If rangefunc is true, *head is the head of the atomic linked list
  	// during a range-over-func execution.
  	head *atomic.Pointer[_defer]
  }
  ```

- 错误处理error&panic
  ```go
  type _panic struct {
  	argp unsafe.Pointer // pointer to arguments of deferred call run during panic; cannot move - known to liblink
  	arg  any            // argument to panic
  	link *_panic        // link to earlier panic
  
  	// startPC and startSP track where _panic.start was called.
  	startPC uintptr
  	startSP unsafe.Pointer
  
  	// The current stack frame that we're running deferred calls for.
  	sp unsafe.Pointer
  	lr uintptr
  	fp unsafe.Pointer
  
  	// retpc stores the PC where the panic should jump back to, if the
  	// function last returned by _panic.next() recovers the panic.
  	retpc uintptr
  
  	// Extra state for handling open-coded defers.
  	deferBitsPtr *uint8
  	slotsPtr     unsafe.Pointer
  
  	recovered   bool // whether this panic has been recovered
  	goexit      bool
  	deferreturn bool
  }
  ```

- GC
  Golang的GC和Java的G1大致相当，STW/可达性分析/mark-sweep/三色标记/读写屏障/并发回收这些都有，但是还没有精细到ZGC/shenandoah。

### 工程化
golang对工程化的专注，暴打Java和C++。

- 模块module
  ```go
  go mod init
  go mod add
  go mod tidy
  go mod download
  go mod graph
  go list -m all
  ```
  以上go mod命令都是在自动维护一个go.mod文件。
  ```go
  module example/hello
  
  go 1.23.4
  
  require rsc.io/quote v1.5.2
  
  require (
  	golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c // indirect
  	rsc.io/sampler v1.3.0 // indirect
  )
  ```

- 编译build
  直接编译成可执行文件，大小和C++的差不多。
  ```go
  go build
  go run
  ```

至此，可以去看K8S源码了。