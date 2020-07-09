# 用BPF实现用户态tracing

BPF是最近Linux内核领域热门的技术。传统的BPF指的是`tcpdump`命令用于过滤网络包的工具，现在BPF已经得到极大的扩展，不再是Berkeley Packet Filter的缩写对应的简单的网络包过滤工具。
从Kernel 4.9之后，BPF已经成为一个完善的内核扩展工具，BPF在内核里运行一个sandbox，用于执行BPF的字节码（bytecode），
在执行BPF程序前，BPF的检查器会对BPF程序的字节码进行安全检查（比如，指针要先判断不为空后再访问，代码里不能有循环，等等），以保证BPF程序不会导致系统崩溃，因为BPF程序执行时是在内核态。
因此，BPF可以很安全地在内核态执行用户编写的程序，而且有安全保证，这比编写内核模块安全太多了。
正是因为BPF能保证安全，并运行在内核态，可以大大简化很多以前很复杂的事情，目前BPF已经应用于性能分析、网络、安全、驱动、区块链等等领域。

已经有很多文章介绍BPF在内核性能分析（Kernel tracing）方面的应用。内核有各种定义的tracepoint用于静态tracing，也可以采用动态tracing来跟踪内核函数调用。
用BPF进行内核tracing，开销小而且性能好，因为BPF程序是运行在内核态，不需要把采集到的数据再传回用户态处理，而是直接在内核态完成数据采集和处理，然后把处理结果传回用户态用于展示。
关于BPF用于内核tracing方面不再赘述，本文专注于利用BPF进行用户态应用性能分析（Userspace tracing）方面。

## 用户态tracing

在进行用户态tracing前，要在程序里定义tracepoints。这里主要介绍Userland Statically Defined Tracepoints（USDT）。

因为USDT依赖`systemtap-sdt-dev`包，先要安装依赖包，以Ubuntu为例，运行`sudo apt install systemtap-sdt-dev`进行安装。
下面的示例`test-server.c`给出如何使用宏`DTRACE_PROBE1`在用户程序里定义tracepoint：
```
#include <sys/sdt.h>
#include <unistd.h>

int main(int argc, char **argv)
{
    int idx = 0;
    while(1) {
        idx++;
        // 自定义的tracepoint
        DTRACE_PROBE1(test_grp, test_idx, idx);
        sleep(1);
    }
    return 0;
}
```
上面的例子，用宏`DTRACE_PROBE1`定义了一个组名为`test_grp`、名称为`test_idx`的用户态tracepoint。该tracepoint只有一个参数，该参数是一个递增的整数变量。
如果要定义有两个或更多参数的tracepoint，要用`DTRACE_PROBE2`、`DTRACE_PROBE3`，以此类推。如果tracepoint不带参数，则用`DTRACE_PROBE`来定义。
用gcc命令编译上面的程序`gcc -g -fno-omit-frame-pointer -O0 test-server.c -o test-server`，得到可执行的二进制程序`test-server`。

### USDT的原理

用宏`DTRACE_PROBE1`定义的tracepoint在编译完成的二进制里对应CPU指令`nop`操作，可以用`gdb`看下二进制文件`test-server`对应的汇编：
```
$ gdb test-server
......
Reading symbols from test-server...
(gdb) disas main <<== gdb命令用于查看二进制文件的汇编
Dump of assembler code for function main:
   0x0000000000001149 <+0>:	endbr64
   0x000000000000114d <+4>:	push   %rbp
   0x000000000000114e <+5>:	mov    %rsp,%rbp
   0x0000000000001151 <+8>:	sub    $0x20,%rsp
   0x0000000000001155 <+12>:	mov    %edi,-0x14(%rbp)
   0x0000000000001158 <+15>:	mov    %rsi,-0x20(%rbp)
   0x000000000000115c <+19>:	movl   $0x0,-0x4(%rbp)
   0x0000000000001163 <+26>:	addl   $0x1,-0x4(%rbp)
   0x0000000000001167 <+30>:	nop <<= tracepoint对应的nop指令
   0x0000000000001168 <+31>:	mov    $0x1,%edi
   0x000000000000116d <+36>:	callq  0x1050 <sleep@plt>
   0x0000000000001172 <+41>:	jmp    0x1163 <main+26>
End of assembler dump.
```
从上面的汇编可以看出，`nop`操作只是放空一个CPU cycle，这个代价非常低，所以程序里定义USDT对性能的影响可以忽略。

在运行时，如果用BPF对`test-server`进行tracing（具体tracing的细节下文会讲），再用`sudo gdb`查看运行时的程序对应的汇编，会发现`nop`操作被替换成`int3`指令了：
```
$ sudo gdb -p $(pidof test-server)
......
(gdb) disas main
Dump of assembler code for function main:
   0x00005583ee599149 <+0>:	endbr64
   0x00005583ee59914d <+4>:	push   %rbp
   0x00005583ee59914e <+5>:	mov    %rsp,%rbp
   0x00005583ee599151 <+8>:	sub    $0x20,%rsp
   0x00005583ee599155 <+12>:	mov    %edi,-0x14(%rbp)
   0x00005583ee599158 <+15>:	mov    %rsi,-0x20(%rbp)
   0x00005583ee59915c <+19>:	movl   $0x0,-0x4(%rbp)
   0x00005583ee599163 <+26>:	addl   $0x1,-0x4(%rbp)
   0x00005583ee599167 <+30>:	int3 <<== tracepoint对应的int3指令
   0x00005583ee599168 <+31>:	mov    $0x1,%edi
   0x00005583ee59916d <+36>:	callq  0x5583ee599050 <sleep@plt>
   0x00005583ee599172 <+41>:	jmp    0x5583ee599163 <main+26>
End of assembler dump.
```
从上面`gdb`展示的运行时`test-server`的汇编，发现`nop`指令被替换为`int3`指令，该指令是设置断点，用于转向执行BPF等tracing工具的指令，诸如采集tracepoint数据等等。
所以USDT是从指令层面进行tracing，对程序运行时性能影响很小。

### 用`bpftrace`命令进行tracing

这里先介绍`bpftrace`命令，这个命令不需要写BPF程序，只用写脚本的方式来实现tracing。在Ubuntu上运行`sudo apt-get install -y bpftrace`来安装`bpftrace`。
运行如下`bpftrace`命令，必须用sudo权限：
```
$ sudo bpftrace \
    -e 'usdt:/home/pwang/usdt_test/test-server:test_grp:test_idx \
        { printf("%d\n", arg0); }' \
    -p $(pidof test-server)
Attaching 1 probe...
1365
1366
1367
1368
1369
1370
^C
```
上述`bpftrace`命令采集到`test-server`的名为`test_idx`的tracepoint，并打印出该tracepoint的第一个参数的值。
该`bpftrace`命令包含如下几个组成部分：
* `-e`的含义是运行后面跟着的脚本，该脚本包含两部分：
    * 第一部分`usdt:/home/pwang/usdt_test/test-server:test_grp:test_idx`是用冒号分隔的四段：
        * `usdt`是tracing类型；
        * `/home/pwang/usdt_test/test-server`是被trace的程序的绝对路径；
        * `test_grp`是tracepoint的组名，也可以省略不写；
        * `test_idx`是tracepoint的名称。
    * 第二部分`{ printf("%d\n", arg0); }`是打印该tracepoint的第一个参数的值；
* `-p`是指定被trace的进程ID。

### 用BCC工具编写BPF程序进行tracing

BCC是用于编写BPF程序的开发框架和编译器。目前BPF程序主要用C语言来写，但是不支持C的全部语法，比如不支持循环。
BCC调用LLVM把BPF的C代码转成BPF的字节码，然后经过BPF检查器的检验，检验通过的BPF字节码成功加载到sandbox里运行。
BCC还支持用Python代码来跟运行在sandbox里的BPF程序进行交互，比如拿到BPF的tracing结果并展示。
在Ubuntu上运行`sudo apt-get install -y bpfcc-tools`来安装BCC。

下面是一段BCC的Python脚本，内嵌了BPF的C代码：
```
#! /usr/bin/python3

from bcc import BPF, USDT
import sys

// 内嵌的BPF的C代码
bpf_src = """
int trace_udst(struct pt_regs *ctx) {
    u32 idx;
    bpf_usdt_readarg(1, ctx, &idx);
    bpf_trace_printk("test_idx=%d tracepoint cachted\\n", idx);
    return 0;
};
"""

// 指定USDT的tracepoint，并关联BPF程序里的函数
procid = int(sys.argv[1])
u = USDT(pid=procid)
u.enable_probe(probe="test_idx", fn_name="trace_udst")

// 加载BPF程序并打印输出结果
b = BPF(text=bpf_src, usdt_contexts=[u])
print("Start USDT tracing")
b.trace_print()
```
上面的BCC代码分成几个部分：
* 第一部分是内嵌的BPF的C代码，用字符串的方式定义给`bpf_src`变量。`bpf_src`里的C代码定义了`trace_udst`函数，该函数的输入参数是抓取到的tracepoint上下文，该函数做两件事：
    * `bpf_usdt_readarg(1, ctx, &idx);`用于从tracepoint上下文中读取tracepoint的第一个参数的值；
    * `bpf_trace_printk("test_idx=%d tracepoint cachted\\n", idx);`打印输出结果，打印的结果会发送到一个管道，管道的路径是`/sys/kernel/debug/tracing/trace_pipe`；
* 第二部分是指定USDT的tracepoint，并关联BPF程序以实现tracing：
    * `u = USDT(pid=procid)`生成USDT对象；
    * `u.enable_probe(probe="test_idx", fn_name="trace_udst")`指定tracepoint为`text_idx`，并指定BPF程序中的`trace_udst`函数为trace到该point的执行操作；
* 第三部分是加载BPF程序并从管道中取出输出结果并打印到STDOUT：
    * `b = BPF(text=bpf_src, usdt_contexts=[u])`生成BPF对象，并关联USDT对象，加载BPF程序到sandbox进行检验和运行；
    * `b.trace_print()`打印每次BPF程序trace到`text_idx`的输出结果。

下面是该BCC代码的执行结果，注意该脚本用`python3`来运行：
```
$ sudo ./ebpf_hello_usdt.py $(pidof test-server)
Start USDT tracing
b'     test-server-27125 [000] .... 26587.633948: 0: test_idx=2958 tracepoint cachted'
b'     test-server-27125 [000] .... 26588.634090: 0: test_idx=2959 tracepoint cachted'
b'     test-server-27125 [000] .... 26589.634237: 0: test_idx=2960 tracepoint cachted'
b'     test-server-27125 [000] .... 26590.634386: 0: test_idx=2961 tracepoint cachted'
b'     test-server-27125 [000] .... 26591.634563: 0: test_idx=2962 tracepoint cachted'
b'     test-server-27125 [000] .... 26592.634714: 0: test_idx=2963 tracepoint cachted'
b'     test-server-27125 [000] .... 26593.634870: 0: test_idx=2964 tracepoint cachted'
```
可以看到BPF程序成功trace到递增的`text_idx`的值。

BPF是非常强大的内核扩展工具，有了BPF可以做很多以前很复杂或做不到的事情。此外，有了BCC，编写BPF程序的复杂度大大降低，用BCC可以写出非常强大的内核扩展应用，而且安全性和性能有保证。
业内人士普遍认为，有了BPF之后，Linux的内核会越来越成为微内核，很多在内核里或用内核扩展来做的事情可以让用户自行编写BPF程序来完成。
