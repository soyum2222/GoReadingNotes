## Golang 获取 IP 寄存器内容
  
  之前用C写过一个调度器，想用Go也写一次，正好最近在学习Plan9汇编，趁机巩固一下。
  
  先理清楚基本思路。因为不能直接对IP寄存器做操作，所以无法直接通过 MOVQ IP，AX 这种方式得到IP的值。
  
  但是总所周知，CALL 指令等价于 PUSH IP ，jmp XXX 。所以我们可以通过CALL指令过通过SP寄存器拿到IP的值。
  
  基本思路很简单
  
  但是和C语言有一些细微的差别
  
  
  C版本
```
int64 getrip() {
    register int64 ip;
    asm volatile (
    "movq 0x10(%%rsp), %0\n\t"
    :"=r"(ip)
    );
    return ip;
}

```


  Go plan9 版本  
```
TEXT ·getip(SB),$0
	MOVQ (SP) ,AX
	MOVQ AX,ret+0(FP)
	RET
```
  
  C版本要比GO版本SP多偏移一个8字节。
  
  为啥呢?  我照着C版本SP偏移8字节取拿IP，结果就是debug了一个下午。
  
  反编译两个版本看一下
  
  
  C
  ```
  getrip:
.LFB0:
        .cfi_startproc
        pushq   %rbp
        .cfi_def_cfa_offset 16
        .cfi_offset 6, -16
        movq    %rsp, %rbp
        .cfi_def_cfa_register 6
        pushq   %rbx
        .cfi_offset 3, -24
#APP
# 9 "main.c" 1
        movq 0x10(%rsp), %rax

# 0 "" 2
#NO_APP
        movq    %rax, %rbx
        movq    %rbx, %rax
        popq    %rbx
        popq    %rbp
        .cfi_def_cfa 7, 8
        ret
        .cfi_endproc
  ```



  GO
  ```
  TEXT main.getip(SB) /mnt/d/gopath/src/touchingfish/plan9/getip/getip.s
        getip.s:4       0x4addf0*       488b442408      mov rax, qword ptr [rsp+0x8]
        getip.s:5       0x4addf5        4889442408      mov qword ptr [rsp+0x8], rax
        getip.s:6       0x4addfa        c3              ret
  ```
  
  看到是因为C在调用方法的时候，因为是内嵌的汇编，所以他会有正常构建栈的过程，开头比go多了一个push %rbp。
  Go中如果是手写汇编方法，这一部分是需要在TEXT .getip(SB),$0-0  这个开后指定栈的大小。我指定的是0 所以SP处就是IP的位子。
  
  值得注意的是，如果把TEXT .getip(SB),$0-0 改成了TEXT .getip(SB),$8-0，这个地方的偏移量又会变成16，不但栈扩充8字节 还有个push BP的过程，又是8字节。(((φ(◎ロ◎;)φ)))
  
  ```
  TEXT main.getip(SB) /mnt/d/gopath/src/touchingfish/plan9/getip/getip.s  
        getip.s:3       0x4addf0*       4883ec10        sub rsp, 0x10
        getip.s:3       0x4addf4        48896c2408      mov qword ptr [rsp+0x8], rbp
        getip.s:3       0x4addf9        488d6c2408      lea rbp, ptr [rsp+0x8]
        getip.s:4       0x4addfe        488b442408      mov rax, qword ptr [rsp+0x8]
        getip.s:5       0x4ade03        4889442418      mov qword ptr [rsp+0x18], rax
        getip.s:6       0x4ade08        488b6c2408      mov rbp, qword ptr [rsp+0x8]
        getip.s:1       0x4ade0d        4883c410        add rsp, 0x10
        getip.s:1       0x4ade11        c3              ret
  ```
  
  
  其实从这个地方可以得到一个信息 ，FP伪寄存器其实是不在当前方法栈内的。他在caller的栈内。
  
  其实这里也可以改成
  
  ```
  TEXT ·getip(SB),$0x8-0
	MOVQ ip-8(FP) ,AX
	MOVQ AX,ret+0(FP)
	RET
  ```
  
  FP伪寄存器是指向IP的位置前8位的，也就是返回值和参数的地址的末尾。这里 推荐 https://xargin.com/plan9-assembly/ 曹大的文章
  
