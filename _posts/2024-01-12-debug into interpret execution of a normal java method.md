---
layout: post
title:  "单步调试java方法在hotspot中的解释执行"
date:   2024-01-12
catalog: true
tags:
    - jvm
    - java
    - hotspot
    - gdb
    - debug
---

## 0.目的

环境：[openjdk-8b120](https://github.com/aristotle0x01/openjdk)，不再是openjdk8u，因为github官方仓库没有，为避免不必要的上传等麻烦，故切换版本。

* java方法是如何解释执行的
* 相邻字节码如何关联起来
* 栈顶缓存是怎么回事
* 栈帧是怎样的
* 如何用gdb一步步调试进入上述关键节点

应该说，本文的目的不是讲明白解释执行的原理，而是调试其关键节点和查验相关物理状态。Debug少不得先阅读源码，读相关博文或者书籍，对大体逻辑已经明白，否则是做不到的[^1] [^2] [^3] [^4] [^5] [^6] [^7] [^8]。为此，验证场景非常简化，java main方法里`int i=1`即返回，这样关涉的字节码类型也很少。

## 1.测试代码

一个简单的静态java main方法:

```java
public class Interpret {
    public static void main(String[] args) {
        int i = 1;
    }
}
```

编译后节码结果：

```java
// javap -verbose Interpret
public static void main(java.lang.String[]);
     descriptor: ([Ljava/lang/String;)V
     flags: ACC_PUBLIC, ACC_STATIC
     Code:
       stack=1, locals=2, args_size=1
          0: iconst_1
          1: istore_1
          2: return
```

## 2.字节码的汇编实现

```shell
/var/shared/openjdk/build/linux-x86_64-normal-server-slowdebug/jdk/bin/java -XX:+PrintInterpreter -XX:LogFile=interpret.log Interpret
```

字节码虚拟指令在hotspot中是通过手写汇编实现的(*hotspot/src/cpu/x86/vm/templateTable_x86_64.cpp*)，在x86_64下输出如下：

```assembly
iconst_1  4 iconst_1  [0x00007f682502c4c0, 0x00007f682502c520]  96 bytes

  0x00007f682502c4c0: push   %rax
  0x00007f682502c4c1: jmpq   0x00007f682502c4f0
  0x00007f682502c4c6: sub    $0x8,%rsp
  0x00007f682502c4ca: vmovss %xmm0,(%rsp)
  0x00007f682502c4cf: jmpq   0x00007f682502c4f0
  0x00007f682502c4d4: sub    $0x10,%rsp
  0x00007f682502c4d8: vmovsd %xmm0,(%rsp)
  0x00007f682502c4dd: jmpq   0x00007f682502c4f0
  0x00007f682502c4e2: sub    $0x10,%rsp
  0x00007f682502c4e6: mov    %rax,(%rsp)
  0x00007f682502c4ea: jmpq   0x00007f682502c4f0
  0x00007f682502c4ef: push   %rax
  0x00007f682502c4f0: mov    $0x1,%eax
  0x00007f682502c4f5: movzbl 0x1(%r13),%ebx
  0x00007f682502c4fa: inc    %r13
  0x00007f682502c4fd: movabs $0x7f683b574ca0,%r10
  0x00007f682502c507: jmpq   *(%r10,%rbx,8)
  0x00007f682502c50b: nop
  ......

----------------------------------------------------------------------
istore_1  60 istore_1  [0x00007f682502e920, 0x00007f682502e960]  64 bytes

  0x00007f682502e920: mov    (%rsp),%eax
  0x00007f682502e923: add    $0x8,%rsp
  0x00007f682502e927: mov    %eax,-0x8(%r14)
  0x00007f682502e92b: movzbl 0x1(%r13),%ebx
  0x00007f682502e930: inc    %r13
  0x00007f682502e933: movabs $0x7f683b5774a0,%r10
  0x00007f682502e93d: jmpq   *(%r10,%rbx,8)
  0x00007f682502e941: nop
  ......
```

上面仅截选最相关指令

## 3.java main方法入口及栈帧debug过程

```shell
gdb /var/shared/openjdk/build/linux-x86_64-normal-server-slowdebug/jdk/bin/java

// jdk/src/share/bin/java.c
// 首先打且仅打该断点，因为进入main方法前也会多次进入下一个断点
b java.c:472 // breakpoint 1

// 启动java测试代码
run Interpret

// 到breakpoint 1后打breakpoint 2
b javaCalls.cpp:393	// breakpoint 2

p 'StubRoutines::_call_stub_entry'
$2 = (address) 0x7fffe1000564

// 可与2紧接着breakpoint 3
b *0x7fffe1000564 // breakpoint 3

// 到breakpoint 3时stepi进入
stepi 

// 再进入_call_stub_entry关键代码，__ call(c_rarg1)(实际为callq *%rsi)即为解释器入口
// p /x $rsi => 0x7fffe101e2e0
// p 'AbstractInterpreter::_entry_table'[0] => 0x7fffe101e2e0 // zerolocals类型方法入口
b *0x7fffe101e2e0 // breakpoint 4

// gdb指令汇编和源码窗口
ctrl+x+a 
ctrl+x+1
ctrl+x+2
ctrl+x+a enter 退出
tui reg general // 查看寄存器
```

### 3.1 breakpoint 1

<u>jdk/src/share/bin/java.c</u>，c++到java main入口。首先在此处断点仅为过滤掉在这之前对breakpoint 2处的调用造成的停顿。

```c++
int JNICALL
JavaMain(void * _args)
{	
  	......
    mainID = (*env)->GetStaticMethodID(env, mainClass, "main",
                                       "([Ljava/lang/String;)V");

    /* Build platform specific argument array */
    mainArgs = CreateApplicationArgs(env, argv, argc);
    CHECK_EXCEPTION_NULL_LEAVE(mainArgs);

    /* Invoke main method. */
    (*env)->CallStaticVoidMethod(env, mainClass, mainID, mainArgs); // line 472

    ret = (*env)->ExceptionOccurred(env) == NULL ? 0 : 1;
    LEAVE();
}
```

### 3.2 breakpoint 2

<u>hotspot/src/share/vm/runtime/javaCalls.cpp</u>，c++调用java方法前的一些信息准备等

```c++
void JavaCalls::call_helper(JavaValue* result, methodHandle* m, JavaCallArguments* args, TRAPS) {
  ......
  address entry_point = method->from_interpreted_entry();
  if (JvmtiExport::can_post_interpreter_events() && thread->is_interp_only_mode()) {
    entry_point = method->interpreter_entry();
  }
  
  // do call
  { JavaCallWrapper link(method, receiver, result, CHECK);
    { HandleMark hm(thread);  

      StubRoutines::call_stub()(	// line 393
        (address)&link,
        // (intptr_t*)&(result->_value), 
        result_val_address,          
        result_type,
        method(),
        entry_point,
        args->parameters(),
        args->size_of_parameters(),
        CHECK
      );

      result = link.result();
    }
  } 
}
```

**StubRoutines::call_stub()**返回了**StubRoutines::_call_stub_entry**，后者其实是一个函数指针，是c++调用java方法的真正入口。

### 3.3 breakpoint 3

该函数就是<u>hotspot/src/cpu/x86/vm/stubGenerator_x86_64.cpp</u>::`generate_call_stub(address& return_address)`，也是人工用汇编写的，主要完成方法栈帧准备，模版解释器入口调用。

```c++
address generate_call_stub(address& return_address) {
    StubCodeMark mark(this, "StubRoutines", "call_stub");
    address start = __ pc();

    // same as in generate_catch_exception()!
    const Address rsp_after_call(rbp, rsp_after_call_off * wordSize); 

    // rbp： CONSTANT_REGISTER_DECLARATION(Register, rbp,    (5))
    // class Address: hotspot/src/cpu/x86/vm/assembler_x86.hpp
    const Address call_wrapper  (rbp, call_wrapper_off   * wordSize);
    const Address result        (rbp, result_off         * wordSize);
    const Address result_type   (rbp, result_type_off    * wordSize);
    const Address method        (rbp, method_off         * wordSize);
    const Address entry_point   (rbp, entry_point_off    * wordSize);
    const Address parameters    (rbp, parameters_off     * wordSize);
    const Address parameter_size(rbp, parameter_size_off * wordSize);

    // same as in generate_catch_exception()!
    const Address thread        (rbp, thread_off         * wordSize);

    const Address r15_save(rbp, r15_off * wordSize);
    const Address r14_save(rbp, r14_off * wordSize);
    const Address r13_save(rbp, r13_off * wordSize);
    const Address r12_save(rbp, r12_off * wordSize);
    const Address rbx_save(rbp, rbx_off * wordSize);

    // stub code
    __ enter();
    __ subptr(rsp, -rsp_after_call_off * wordSize);
  	......
      
    // call Java function
    __ BIND(parameters_done);
    __ movptr(rbx, method);             // get Method*
    __ movptr(c_rarg1, entry_point);    // get entry_point
    __ mov(r13, rsp);                   // set sender sp
    BLOCK_COMMENT("call Java function");
    __ call(c_rarg1);

    BLOCK_COMMENT("call_stub_return_address:");
    return_address = __ pc();
    ......
}
```

文件头部有一段栈帧的描述，非常重要：

```c++
// Call stubs are used to call Java from C
  //
  // Linux Arguments:
  //    c_rarg0:   call wrapper address                   address
  //    c_rarg1:   result                                 address
  //    c_rarg2:   result type                            BasicType
  //    c_rarg3:   method                                 Method*
  //    c_rarg4:   (interpreter) entry point              address
  //    c_rarg5:   parameters                             intptr_t*
  //    16(rbp): parameter size (in words)              int
  //    24(rbp): thread                                 Thread*
  //
  //     [ return_from_Java     ] <--- rsp
  //     [ argument word n      ]
  //      ...
  // -12 [ argument word 1      ]
  // -11 [ saved r15            ] <--- rsp_after_call
  // -10 [ saved r14            ]
  //  -9 [ saved r13            ]
  //  -8 [ saved r12            ]
  //  -7 [ saved rbx            ]
  //  -6 [ call wrapper         ]
  //  -5 [ result               ]
  //  -4 [ result type          ]
  //  -3 [ method               ]
  //  -2 [ entry point          ]
  //  -1 [ parameters           ]
  //   0 [ saved rbp            ] <--- rbp
  //   1 [ return address       ]
  //   2 [ parameter size       ]
  //   3 [ thread               ]
```

结合源码，上述描述，即可对照理解下面gdb里的汇编代码：

![image](https://github.com/aristotle0x01/aristotle0x01.github.io/assets/2216435/ae85e812-b61c-446d-837a-f6473276dfac)

### 3.4 breakpoint 4

在上述breakpoint 3中，单步执行到

```assembly
0x7fffe100066f:	callq  *%rsi
```

![image](https://github.com/aristotle0x01/aristotle0x01.github.io/assets/2216435/d63ca5a2-bf53-4129-9adc-e6bc40766736)

<u>hotspot/src/cpu/x86/vm/templateInterpreter_x86_64.cpp</u>，**generate_normal_entry**准备局部变量，java栈帧，并跳转到第一条字节码开始执行：

```c++
//
// Generic interpreted method entry to (asm) interpreter
//
address InterpreterGenerator::generate_normal_entry(bool synchronized) {
  // determine code generation flags
  bool inc_counter  = UseCompiler || CountCompiledCalls;

  // ebx: Method*
  // r13: sender sp
  address entry_point = __ pc();

  const Address constMethod(rbx, Method::const_offset());
  const Address access_flags(rbx, Method::access_flags_offset());
  const Address size_of_parameters(rdx,
                                   ConstMethod::size_of_parameters_offset());
  const Address size_of_locals(rdx, ConstMethod::size_of_locals_offset());


  // get parameter size (always needed)
  // note: rdx as a local var set here kind of violate intuition, but not set 
  //  before size_of_parameters. It is tested ok to move "__ movptr(rdx, constMethod)" before size_of_parameters
  __ movptr(rdx, constMethod);
  __ load_unsigned_short(rcx, size_of_parameters);

  // rbx: Method*
  // rcx: size of parameters
  // r13: sender_sp (could differ from sp+wordSize if we were called via c2i )

  __ load_unsigned_short(rdx, size_of_locals); // get size of locals in words
  __ subl(rdx, rcx); // rdx = no. of additional locals

  // see if we've got enough room on the stack for locals plus overhead.
  generate_stack_overflow_check();

  // get return address
  __ pop(rax);

  // compute beginning of parameters (r14)
  __ lea(r14, Address(rsp, rcx, Address::times_8, -wordSize));

  // rdx - # of additional locals
  // allocate space for locals
  // explicitly initialize locals
  {
    Label exit, loop;
    __ testl(rdx, rdx);
    __ jcc(Assembler::lessEqual, exit); // do nothing if rdx <= 0
    __ bind(loop);
    __ push((int) NULL_WORD); // initialize local variables
    __ decrementl(rdx); // until everything initialized
    __ jcc(Assembler::greater, loop);
    __ bind(exit);
  }

  // initialize fixed part of activation frame
  generate_fixed_frame(false);
  ......
  __ dispatch_next(vtos);
  
  return entry_point;
}
```

![image](https://github.com/aristotle0x01/aristotle0x01.github.io/assets/2216435/e2054e0e-1424-4872-bf95-9ae1b0fa7db4)

## 4.字节码解释执行过程

```shell
gdb /var/shared/openjdk/build/linux-x86_64-normal-server-slowdebug/jdk/bin/java

// 得到iconst_1 istore_1解释执行的汇编代码入口点
run -XX:+PrintInterpreter Interpret | grep -E -A 20 'iconst_1|istore_1'
```

结果如下：

```assembly
iconst_1  4 iconst_1  [0x00007fffe102c4c0, 0x00007fffe102c520]  96 bytes

  0x00007fffe102c4c0: push   %rax
  0x00007fffe102c4c1: jmpq   0x00007fffe102c4f0
  0x00007fffe102c4c6: sub    $0x8,%rsp
  0x00007fffe102c4ca: vmovss %xmm0,(%rsp)
  0x00007fffe102c4cf: jmpq   0x00007fffe102c4f0
  0x00007fffe102c4d4: sub    $0x10,%rsp
  0x00007fffe102c4d8: vmovsd %xmm0,(%rsp)
  0x00007fffe102c4dd: jmpq   0x00007fffe102c4f0
  0x00007fffe102c4e2: sub    $0x10,%rsp
  0x00007fffe102c4e6: mov    %rax,(%rsp)
  0x00007fffe102c4ea: jmpq   0x00007fffe102c4f0
  0x00007fffe102c4ef: push   %rax
  0x00007fffe102c4f0: mov    $0x1,%eax
  0x00007fffe102c4f5: movzbl 0x1(%r13),%ebx
  0x00007fffe102c4fa: inc    %r13
  0x00007fffe102c4fd: movabs $0x7ffff73b9ca0,%r10
  0x00007fffe102c507: jmpq   *(%r10,%rbx,8)
  0x00007fffe102c50b: nop
  0x00007fffe102c50c: nop
--
istore_1  60 istore_1  [0x00007fffe102e920, 0x00007fffe102e960]  64 bytes

  0x00007fffe102e920: mov    (%rsp),%eax
  0x00007fffe102e923: add    $0x8,%rsp
  0x00007fffe102e927: mov    %eax,-0x8(%r14)
  0x00007fffe102e92b: movzbl 0x1(%r13),%ebx
  0x00007fffe102e930: inc    %r13
  0x00007fffe102e933: movabs $0x7ffff73bc4a0,%r10
  0x00007fffe102e93d: jmpq   *(%r10,%rbx,8)
  0x00007fffe102e941: nop
  0x00007fffe102e942: nop
```

接着断点：

```shell
// 先执行到main
b java.c:472

run Interpret
```

![image](https://github.com/aristotle0x01/aristotle0x01.github.io/assets/2216435/dabb2a6e-1395-4670-8a6a-997bee579975)

接着单步执行到`0x7fffe102c507:	jmpq   *(%r10,%rbx,8)`

![image](https://github.com/aristotle0x01/aristotle0x01.github.io/assets/2216435/39b5f4b8-2c4c-460a-b05b-ba4644b96195)

<u>hotspot/src/share/vm/interpreter/templateTable.cpp</u>

![image](https://github.com/aristotle0x01/aristotle0x01.github.io/assets/2216435/ad435adb-30c0-42ef-bb75-541bb7c6aaaa)

负责字节码跳转的汇编代码在<u>hotspot/src/cpu/x86/vm/interp_masm_x86_64.cpp</u>

```c++
void InterpreterMacroAssembler::dispatch_next(TosState state, int step) {
  // load next bytecode (load before advancing r13 to prevent AGI)
  load_unsigned_byte(rbx, Address(r13, step));
  // advance r13
  increment(r13, step);
  dispatch_base(state, Interpreter::dispatch_table(state));
}

void InterpreterMacroAssembler::dispatch_base(TosState state,
                                              address* table,
                                              bool verifyoop) {
  ......
  lea(rscratch1, ExternalAddress((address)table));
  jmp(Address(rscratch1, rbx, Address::times_8));
}
```

## 5.references

[^1]: [Brief Intro to the Template Interpreter in OpenJDK](https://albertnetymk.github.io/2021/08/03/template_interpreter/) - 勾勒了字节码解释执行及关联跳转原理
[^2]: [Open Heart Surgery:  Analyzing and Debugging the HotSpot VM at the OS Level](https://www.progdoc.de/papers/JavaOne2014/javaone2014_con3138.html#(1)) - gdb debug hotspot关键技巧
[^3]: [揭秘Java虚拟机:JVM设计原理与实现](https://book.douban.com/subject/27086821/) - 书籍：详细讲解了解释执行过程和栈帧变化
[^4]: [[讨论]HotSpot 解释器是怎样执行bytecode 的](https://hllvm-group.iteye.com/group/topic/40491) - 十年前的精彩讨论
[^5]: [Debugging the OpenJDK JVM interpreter in Linux x86_64](https://martin.uy/blog/debugging-hotspot-openjdk-jvm-x86_64-interpreter-in-linux/)
[^6]: [OpenJDK 8 interpreter debug](https://stackoverflow.com/questions/68391777/openjdk-8-interpreter-debug)
[^7]: [How native code is converted into machine code in jvm](https://stackoverflow.com/questions/56823223/how-native-code-is-converted-into-machine-code-in-jvm)
[^8]: [专注虚拟机与编译器研究](https://www.cnblogs.com/mazhimazhi/)
