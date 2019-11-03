## defer 源码阅读 （go 1.13）
     
     直接上代码
  
 ```$xslt

package main

import "fmt"

func main() {
	defer func() {
		fmt.Println("defer")
	}()
}

```

    反编译
    
```
"".main STEXT size=113 args=0x0 locals=0x48
	0x0000 00000 (defer.go:5)	TEXT	"".main(SB), ABIInternal, $72-0
	0x0000 00000 (defer.go:5)	MOVQ	TLS, CX
	0x0009 00009 (defer.go:5)	MOVQ	(CX)(TLS*2), CX
	0x0010 00016 (defer.go:5)	CMPQ	SP, 16(CX)
	0x0014 00020 (defer.go:5)	JLS	106
	0x0016 00022 (defer.go:5)	SUBQ	$72, SP
	0x001a 00026 (defer.go:5)	MOVQ	BP, 64(SP)
	0x001f 00031 (defer.go:5)	LEAQ	64(SP), BP
	0x0024 00036 (defer.go:5)	FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0024 00036 (defer.go:5)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0024 00036 (defer.go:5)	FUNCDATA	$2, gclocals·9fb7f0986f647f17cb53dda1484e0f7a(SB)
	0x0024 00036 (defer.go:6)	PCDATA	$0, $0
	0x0024 00036 (defer.go:6)	PCDATA	$1, $0
	0x0024 00036 (defer.go:6)	MOVL	$0, ""..autotmp_1+8(SP)
	0x002c 00044 (defer.go:6)	PCDATA	$0, $1
	0x002c 00044 (defer.go:6)	LEAQ	"".main.func1·f(SB), AX
	0x0033 00051 (defer.go:6)	PCDATA	$0, $0
	0x0033 00051 (defer.go:6)	MOVQ	AX, ""..autotmp_1+32(SP)
	0x0038 00056 (defer.go:6)	PCDATA	$0, $1
	0x0038 00056 (defer.go:6)	LEAQ	""..autotmp_1+8(SP), AX
	0x003d 00061 (defer.go:6)	PCDATA	$0, $0
	0x003d 00061 (defer.go:6)	MOVQ	AX, (SP)
	0x0041 00065 (defer.go:6)	CALL	runtime.deferprocStack(SB)
	0x0046 00070 (defer.go:6)	TESTL	AX, AX
	0x0048 00072 (defer.go:6)	JNE	90
	0x004a 00074 (defer.go:9)	XCHGL	AX, AX
	0x004b 00075 (defer.go:9)	CALL	runtime.deferreturn(SB)
	0x0050 00080 (defer.go:9)	MOVQ	64(SP), BP
	0x0055 00085 (defer.go:9)	ADDQ	$72, SP
	0x0059 00089 (defer.go:9)	RET
	0x005a 00090 (defer.go:6)	XCHGL	AX, AX
	0x005b 00091 (defer.go:6)	CALL	runtime.deferreturn(SB)
	0x0060 00096 (defer.go:6)	MOVQ	64(SP), BP
	0x0065 00101 (defer.go:6)	ADDQ	$72, SP
	0x0069 00105 (defer.go:6)	RET
	0x006a 00106 (defer.go:6)	NOP
	0x006a 00106 (defer.go:5)	PCDATA	$1, $-1
	0x006a 00106 (defer.go:5)	PCDATA	$0, $-1
	0x006a 00106 (defer.go:5)	CALL	runtime.morestack_noctxt(SB)
	0x006f 00111 (defer.go:5)	JMP	0
```
 
    找出defer 相关的 runtime 方法  `deferprocStack`  `deferreturn`，开读
    
    
```$xslt
// deferprocStack queues a new deferred function with a defer record on the stack.
// The defer record must have its siz and fn fields initialized.
// All other fields can contain junk.
// The defer record must be immediately followed in memory by the arguments of the defer.
// Nosplit because the arguments on the stack won't be scanned until the defer record is spliced into the gp._defer list.
//go:nosplit
func deferprocStack(d *_defer) {
	gp := getg()

    // 当前运行的goroutine
	if gp.m.curg != gp {
		// go code on the system stack can't defer
		throw("defer on system stack")
	}
	// siz and fn are already set.
	// The other fields are junk on entry to deferprocStack and
	// are initialized here.
	d.started = false
	d.heap = false
	d.sp = getcallersp()
	d.pc = getcallerpc()
	// The lines below implement:
	//   d.panic = nil
	//   d.link = gp._defer
	//   gp._defer = d
	// But without write barriers. The first two are writes to
	// the stack so they don't need a write barrier, and furthermore
	// are to uninitialized memory, so they must not use a write barrier.
	// The third write does not require a write barrier because we
	// explicitly mark all the defer structures, so we don't need to
	// keep track of pointers to them with a write barrier.
	*(*uintptr)(unsafe.Pointer(&d._panic)) = 0
	*(*uintptr)(unsafe.Pointer(&d.link)) = uintptr(unsafe.Pointer(gp._defer))
	*(*uintptr)(unsafe.Pointer(&gp._defer)) = uintptr(unsafe.Pointer(d))

	return0()
	// No code can go here - the C return register has
	// been set and must not be clobbered.
}


```
deferprocStack 的参数是一个_defer但是我们并没有传，那么这个值是从哪里来的呢？我们回去看下汇编
        

            	MOVL	$0, ""..autotmp_1+8(SP)
            	PCDATA	$0, $1
            	LEAQ	"".main.func1·f(SB), AX
            	PCDATA	$0, $0
            	MOVQ	AX, ""..autotmp_1+32(SP)
            	PCDATA	$0, $1
            	LEAQ	""..autotmp_1+8(SP), AX
            	PCDATA	$0, $0
            	MOVQ	AX, (SP)
            	CALL	runtime.deferprocStack(SB)
        	
  
           
   第五行 AX, ""..autotmp_1+32(SP) 把AX的值放入SP+32这个地址中，AX寄存器可以从第三行看到，
   
   是func1的地址，然后看倒数第二行MOVQ	AX, (SP) 和第一行 MOVL	$0, ""..autotmp_1+8(SP) ，
   
   deferprocStack的参数是一个_defer指针 sp+8 地址就是作为这个结构体参数的首地址。
   
   首地址为SP+8。那么SP+32 func1 方法的地址，偏移量为24。
        
        type _defer struct {
        	siz     int32 // 占用4字节  //偏移0
        	started bool  // 占用1字节  //偏移4
        	heap    bool  // 占用1字节  //偏移6
        	// 内存对齐 这里会补上两个2节
        	sp      uintptr // 64位机器下是8字节 //偏移8
        	pc      uintptr // 64位机器下是8字节 //偏移16
        	fn      *funcval // 64位机器下是8字节 //偏移24 
        	_panic  *_panic  //  64位机器下是8字节
        	link    *_defer  // 64位机器下是8字节
        }
        
  可以发现fn 正好是24偏移字节的位置，根据以上猜测sp+32位子存放的是func地址sp+0存放siz参数。siz参数是哪里来的呢，是sp+8的地址，`sp+8`的地址由第一行可以看到是0。
  
  (这个地方如果使用goland的debug打入断点去跟踪`d.siz`，结果会与汇编不同，猜测是在编译时做了优化，如果用dlv或者gdb工具，debug汇编代码能验证上面的推论。)
  
  搞清楚了这一点，deferprocStack方法的内容就变得索然无味，就是对_defer的其他几个 "junk" 字段做初始化赋值。然后把他放入到当前G中。




deferprocStack搞清楚了我们接下来看deferreturn

```go
// Run a deferred function if there is one.
// The compiler inserts a call to this at the end of any
// function which calls defer.
// If there is a deferred function, this will call runtime·jmpdefer,
// which will jump to the deferred function such that it appears
// to have been called by the caller of deferreturn at the point
// just before deferreturn was called. The effect is that deferreturn
// is called again and again until there are no more deferred functions.
// Cannot split the stack because we reuse the caller's frame to
// call the deferred function.

// The single argument isn't actually used - it just has its address
// taken so it can be matched against pending defers.
//go:nosplit
func deferreturn(arg0 uintptr) {
	gp := getg()
	d := gp._defer
	if d == nil {
		return
	}
	sp := getcallersp()
	if d.sp != sp {
		return
	}

	// Moving arguments around.
	//
	// Everything called after this point must be recursively
	// nosplit because the garbage collector won't know the form
	// of the arguments until the jmpdefer can flip the PC over to
	// fn.
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




注释第一行说明了，这个deferreturn这个方法是编译器自动添加。具体干的事情其实就是把defer执行defer方法，但是它是怎么执行的呢？可以看到最后它调用了`jmpdefer` 方法，这个方法是使用汇编实现的，那么就到了喜闻乐见的读汇编环节了。


```go

// func jmpdefer(fv *funcval, argp uintptr)
// argp is a caller SP.
// called from deferreturn.
// 1. pop the caller
// 2. sub 5 bytes from the callers return
// 3. jmp to the argument
TEXT runtime·jmpdefer(SB), NOSPLIT, $0-16
	MOVQ	fv+0(FP), DX	// fn
	MOVQ	argp+8(FP), BX	// caller sp
	LEAQ	-8(BX), SP	// caller sp after CALL
	MOVQ	-8(SP), BP	// restore BP as if deferreturn returned (harmless if framepointers not in use)
	SUBQ	$5, (SP)	// return to CALL again
	MOVQ	0(DX), BX
	JMP	BX	// but first run the deferred function

```
        
这里需要结合前面反编译生成的汇编来结合阅读，理解上下文。前面反编译的汇编我们只用关注下面几句就行。
 ```go
    0x0024 00036 (defer.go:6)	MOVL	$0, ""..autotmp_1+8(SP)
	...
	0x0038 00056 (defer.go:6)	LEAQ	""..autotmp_1+8(SP), AX
	0x003d 00061 (defer.go:6)	MOVQ	AX, (SP)
	0x0041 00065 (defer.go:6)	CALL	runtime.deferprocStack(SB)
    ...
	0x004b 00075 (defer.go:9)	CALL	runtime.deferreturn(SB)
	0x0050 00080 (defer.go:9)	MOVQ	64(SP), BP
	0x0055 00085 (defer.go:9)	ADDQ	$72, SP
	0x0059 00089 (defer.go:9)	RET
	0x005a 00090 (defer.go:6)	XCHGL	AX, AX
	0x005b 00091 (defer.go:6)	CALL	runtime.deferreturn(SB)
	0x0060 00096 (defer.go:6)	MOVQ	64(SP), BP
```

众所周知`call F`指令是 `push ip,esp` 加上`jump F`指令实现。也就是说，通过`CALL`调用的方法，它的物理SP寄存器就是存放的他caller的IP寄存器。

由上面的代码可知,`0x005b 00091 (defer.go:6)	CALL	runtime.deferreturn(SB)` 这里传入的参数，是SP寄存器，其值为 第一层调用栈的`SP+8`，然后对这个值取地址-8后，得到第一层caller的`SP`,由于他CALL了一个函数，那么对这个`SP`-8 ,得到的是应该是他push进去的IP。这里地方IP应该是00080，然后把他-5，得到00075，这个值刚好是` 0x004b 00075 (defer.go:9)	CALL	runtime.deferreturn(SB)`这条语句的IP。

妙啊，这里是一个递归方法！一直执行到defer没有值为止。

  