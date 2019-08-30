 ## panic and recover 源码阅读
 
 事情起因是因为上班无聊水群（此处领导不可见），发现群里有小伙伴提了个关于 recover 捕获 panic 的问题  
 
 代码1:
 
 ```
 func main() {
 	defer recover_func()
 	panic("panic")
 }
 func recover_func() {
 	func() {
 		err := recover()
 		if err != nil {
 			fmt.Println("recover")
 			fmt.Println(err)
 		}
 	}()
 }
 
 //out put: panic: panic
 ```
 
 代码2 
 ```
 func main() {
 	defer recover_func()
 	panic("panic")
 }
 func recover_func() {
 	err := recover()
 	if err != nil {
 		fmt.Println("recover")
 		fmt.Println(err)
 	}
 }
 //out put: recover
 //         panic
 ```
 
 由上代码可见，代码1无法成功recover捕获panic，而代码2能正常捕获。  
 这成功触及到了我的知识盲区。于是决定探索一下panic和recover的奥秘。  
 
 
 首先我们需要找到panic在go中的源码。
 ```$xslt
 TEXT	"".main(SB), ABIInternal, $24-0
 MOVQ	(TLS), CX
 CMPQ	SP, 16(CX)
 JLS	59
 SUBQ	$24, SP
 MOVQ	BP, 16(SP)
 LEAQ	16(SP), BP
 FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
 FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
 FUNCDATA	$3, gclocals·9fb7f0986f647f17cb53dda1484e0f7a(SB)
 PCDATA	$2, $1
 PCDATA	$0, $0
 LEAQ	type.string(SB), AX
 PCDATA	$2, $0
 MOVQ	AX, (SP)
 PCDATA	$2, $1
 LEAQ	"".statictmp_0(SB), AX
 PCDATA	$2, $0
 MOVQ	AX, 8(SP)
 CALL	runtime.gopanic(SB)
 UNDEF
 NOP
 PCDATA	$0, $-1
 PCDATA	$2, $-1
 CALL	runtime.morestack_noctxt(SB)
 JMP	0
```
 可见panic源码在runtime.gopanic() 这个方法中。
 
 ```$xslt
func gopanic(e interface{}) {
	gp := getg()
	
	....省略代码
	
    //产生一个新的panic
	var p _panic
	p.arg = e
	p.link = gp._panic
	gp._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

	atomic.Xadd(&runningPanicDefers, 1)

	for {
	    // 从当前g获取一个defer
		d := gp._defer
		if d == nil {  //如果当前g不存在defer 那么直接退出
			break
		}

		// If defer was started by earlier panic or Goexit (and, since we're back here, that triggered a new panic),
		// take defer off list. The earlier panic or Goexit will not continue running.
		if d.started { //这里判断这个defer方法是否被执行过 如果被执行过就获取defer中link字段指向的下一个defer
			if d._panic != nil {
				d._panic.aborted = true
			}
			d._panic = nil
			d.fn = nil
			gp._defer = d.link
			freedefer(d)
			continue
		}

		// Mark defer as started, but keep on list, so that traceback
		// can find and update the defer's argument frame if stack growth
		// or a garbage collection happens before reflectcall starts executing d.fn.
		d.started = true //这里把defer状态修改为已经执行

		// Record the panic that is running the defer.
		// If there is a new panic during the deferred call, that panic
		// will find d in the list and will mark d._panic (this panic) aborted.
		d._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

		p.argp = unsafe.Pointer(getargp(0))
		reflectcall(nil, unsafe.Pointer(d.fn), deferArgs(d), uint32(d.siz), uint32(d.siz))
		p.argp = nil

		// reflectcall did not panic. Remove d.
		if gp._defer != d {
			throw("bad defer entry in panic")
		}
		d._panic = nil
		d.fn = nil
		gp._defer = d.link

		// trigger shrinkage to test stack copy. See stack_test.go:TestStackPanic
		//GC()

        //获取当前的pc寄存器
		pc := d.pc
		sp := unsafe.Pointer(d.sp) // must be pointer so it gets adjusted during stack copy
		freedefer(d)
		if p.recovered {//如果这个panic已经被recovered处理了
			atomic.Xadd(&runningPanicDefers, -1)
            //获取此panic的上一个panic
			gp._panic = p.link
			// Aborted panics are marked but remain on the g.panic list.
			// Remove them from the list.
			for gp._panic != nil && gp._panic.aborted {
				gp._panic = gp._panic.link
			}
			if gp._panic == nil { // must be done with signal
				gp.sig = 0
			}
			// Pass information about recovering frame to recovery.
			gp.sigcode0 = uintptr(sp)
			gp.sigcode1 = pc
			mcall(recovery)
			throw("recovery failed") // mcall should not return
		}
	}

	// ran out of deferred calls - old-school panic now
	// Because it is unsafe to call arbitrary user code after freezing
	// the world, we call preprintpanics to invoke all necessary Error
	// and String methods to prepare the panic strings before startpanic.
	preprintpanics(gp._panic)

	fatalpanic(gp._panic) // should not return
	*(*int)(nil) = 0      // not reached
}

```

由上面代码可以了解到，其实panic方法就是向我们当前g赋值了一个_panic结构体。  
看下_panic 结构体
```$xslt
type _panic struct {
	argp      unsafe.Pointer // pointer to arguments of deferred call run during panic; cannot move - known to liblink
	arg       interface{}    // argument to panic
	link      *_panic        // link to earlier panic
	recovered bool           // whether this panic is over
	aborted   bool           // the panic was aborted
}
```

它其实是一个连表结构link元素指向它的上一个_panic。  
阅读了gopanic方法后，大致明白它干了件什么事情。  
在再用panic方法后，会把当前g的defer依次执行，每执行一次就判断一次panic是否被recover处理。如果没有被处理那么就进行下一次循环，直到当前g的defer方法耗尽，defer耗尽后就进行old-school 的panic做法。  
如果在过程中发现panic被recover处理了，那么就执行recovery方法，进行一次上下文跳转，类似于linux内核中进程调度一样的操作，重置PC寄存器的值，跳转到g当前defer方法的PC位子。


那么我们来看下具体处理的recover方法:
```cassandraql
func gorecover(argp uintptr) interface{} {
	// Must be in a function running as part of a deferred call during the panic.
	// Must be called from the topmost function of the call
	// (the function used in the defer statement).
	// p.argp is the argument pointer of that topmost deferred function call.
	// Compare against argp reported by caller.
	// If they match, the caller is the one who can recover.
	gp := getg()
	p := gp._panic
	if p != nil && !p.recovered && argp == uintptr(p.argp) {
		p.recovered = true
		return p.arg
	}
	return nil
}

```

recover方法干了件非常简单的事情，把当前g的_panic中的recovered修改为true，并且返回panic的arg。  
看到这里算是把panic做的事情搞明白了，于是逐渐忘记了小伙伴的问题。  
其实小伙伴的问题是在recover方法中发生的，recover方法需要判断  argp == uintptr(p.argp) ，argp是调用reccover 的方法指针，p.argp则是在gopanic方法中赋值的一个 uintptr(0),而argp参数来源是		`reflectcall(nil, unsafe.Pointer(d.fn), deferArgs(d), uint32(d.siz), uint32(d.siz))` 中的  deferArgs(d) ，在gopanic 中两者相等所以能通过gorecover的判断。但是文章开头的 `代码1` 那么此处的argp参数为func闭包指针，与panic的argp不相等，所以`代码1`无法recover panic。  




