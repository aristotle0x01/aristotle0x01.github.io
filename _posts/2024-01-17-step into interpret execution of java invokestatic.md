---
layout: post
title:  "单步调试java invokestatic在hotspot中的解释执行"
date:   2024-01-17
catalog: true
tags:
    - jvm
    - hotspot
    - invokestatic
    - gdb
    - debug

---

## 0.目的

接上篇，环境及版本等仍相同

* **`invokestatic`**的汇编实现
* 参数传递
* 所谓方法解析(resolve)的内涵

## 1.invokestatic的汇编实现

先大概过一遍相关核心代码，这样便于与最终生成的汇编对照，以助于理解其逻辑：

<u>hotspot/src/cpu/x86/vm/templateTable_x86_64.cpp</u>

```c++
void TemplateTable::invokestatic(int byte_no) {
  transition(vtos, vtos);
  assert(byte_no == f1_byte, "use this argument");
  // rbx寄存器用来接收invokestatic指向的Method*
  prepare_invoke(byte_no, rbx);  // get f1 Method*
  // do the call
  __ profile_call(rax);
  __ profile_arguments_type(rax, rbx, r13, false);
  __ jump_from_interpreted(rbx, rax);
}
```

接着看**prepare_invoke**，逻辑实现于**load_invoke_cp_cache_entry**：

```c++
void TemplateTable::load_invoke_cp_cache_entry(int byte_no,
                                               Register method,
                                               Register itable_index,
                                               Register flags,
                                               bool is_invokevirtual,
                                               bool is_invokevfinal, /*unused*/
                                               bool is_invokedynamic) {
  const Register cache = rcx;
  const Register index = rdx;
  ......
  // 这个地方offset的计算稍微有点绕，需要关注一下ConstantPool，ConstantPoolCache和
  // ConstantPoolCacheEntry类的定义及相互关系。其中ConstantPoolCache的对象分配allocate
  // 方法使用了C++ placement new语义，也就是说提前分配了ConstantPoolCache对象+length*sizeof(ConstantPoolCacheEntry)
  // 长度的内存，然后返回了ConstantPoolCache*。此后通过ConstantPoolCache*就可以以下标方式访问ConstantPoolCacheEntry
  // 元素了.
  // 此外Address(cache, index, Address::times_ptr, method_offset) => mov 0x18(%rcx,%rdx,8),%rbx
  // 相当于说是先通过下标到达大概位置，然后再加上偏移量。而直觉计算是先加上偏移量到达ConstantPoolCacheEntry
  // 数组头部，然后再通过下标索引
  const int method_offset = in_bytes(
    ConstantPoolCache::base_offset() +
      ((byte_no == f2_byte)
       ? ConstantPoolCacheEntry::f2_offset()
       : ConstantPoolCacheEntry::f1_offset()));
  const int flags_offset = in_bytes(ConstantPoolCache::base_offset() +
                                    ConstantPoolCacheEntry::flags_offset());
  // access constant pool cache fields
  const int index_offset = in_bytes(ConstantPoolCache::base_offset() +
                                    ConstantPoolCacheEntry::f2_offset());

  size_t index_size = (is_invokedynamic ? sizeof(u4) : sizeof(u2));
  resolve_cache_and_index(byte_no, cache, index, index_size);
  // 编译为mov 0x18(%rcx,%rdx,8),%rbx，因为rbx作为入参来接收method
    __ movptr(method, Address(cache, index, Address::times_ptr, method_offset));

  if (itable_index != noreg) {
    // pick up itable or appendix index from f2 also:
    __ movptr(itable_index, Address(cache, index, Address::times_ptr, index_offset));
  }
  __ movl(flags, Address(cache, index, Address::times_ptr, flags_offset));
}
```

再看**resolve_cache_and_index**

```c++
void TemplateTable::resolve_cache_and_index(int byte_no,
                                            Register Rcache,
                                            Register index,
                                            size_t index_size) {
  const Register temp = rbx;
  Label resolved;
  	// 路径1: 通过ConstantPoolCache返回
    __ get_cache_and_index_and_bytecode_at_bcp(Rcache, index, temp, byte_no, 1, index_size);
  	// 0xb8即为ByteCodes::_invokestatic
    __ cmpl(temp, (int) bytecode());  // 汇编结果：cmp    $0xb8,%ebx
    // 如果已经完成解析(resolve)，那么直接跳到方法末尾
    __ jcc(Assembler::equal, resolved);

  // 路径2: 方法解析
  // 首次调用，需要完成方法解析，并将Method* 放入ConstantPoolCache相应下标位置保存
  address entry;
  switch (bytecode()) {
  case Bytecodes::_getstatic:
  case Bytecodes::_putstatic:
  case Bytecodes::_getfield:
  case Bytecodes::_putfield:
    entry = CAST_FROM_FN_PTR(address, InterpreterRuntime::resolve_get_put);
    break;
  case Bytecodes::_invokevirtual:
  case Bytecodes::_invokespecial:
  case Bytecodes::_invokestatic:
  case Bytecodes::_invokeinterface:
    entry = CAST_FROM_FN_PTR(address, InterpreterRuntime::resolve_invoke);
    break;
  ......
  default:
    fatal(err_msg("unexpected bytecode: %s", Bytecodes::name(bytecode())));
    break;
  }
  __ movl(temp, (int) bytecode());
  __ call_VM(noreg, entry, temp);

  // Update registers with resolved info
  __ get_cache_and_index_at_bcp(Rcache, index, 1, index_size);
  __ bind(resolved);
}
```

在看路径1: **__ get_cache_and_index_and_bytecode_at_bcp**

<u>hotspot/src/cpu/x86/vm/interp_masm_x86_64.cpp</u>

```c++
void InterpreterMacroAssembler::get_cache_and_index_and_bytecode_at_bcp(Register cache,
                                                                        Register index,
                                                                        Register bytecode,
                                                                        int byte_no,
                                                                        int bcp_offset,
                                                                        size_t index_size) {
  // 取得ConstantPoolCache指针与invokestatic对应方法常量池索引
  get_cache_and_index_at_bcp(cache, index, bcp_offset, index_size);
  // 从ConstantPoolCache后续ConstantPoolCacheEntry数组下标处取得bytecode, 如果此前完成解析则最终应为0xb8(invokestatic)
  movl(bytecode, Address(cache, index, Address::times_ptr, ConstantPoolCache::base_offset() + ConstantPoolCacheEntry::indices_offset()));
  const int shift_count = (1 + byte_no) * BitsPerByte;
  shrl(bytecode, shift_count);
  andl(bytecode, ConstantPoolCacheEntry::bytecode_1_mask);
}
```

先看**get_cache_and_index_at_bcp**

```c++
void InterpreterMacroAssembler::get_cache_and_index_at_bcp(Register cache,
                                                           Register index,
                                                           int bcp_offset,
                                                           size_t index_size) {
  get_cache_index_at_bcp(index, bcp_offset, index_size);
  movptr(cache, Address(rbp, frame::interpreter_frame_cache_offset * wordSize));
  shll(index, 2);
}

void InterpreterMacroAssembler::get_cache_index_at_bcp(Register index,
                                                       int bcp_offset,
                                                       size_t index_size) {
  if (index_size == sizeof(u2)) {
    // 从字节码流中取得short型数值，因为invokestatic后面紧跟着两个字节的方法常量池索引
    load_unsigned_short(index, Address(r13, bcp_offset));
  } else if (index_size == sizeof(u4)) {
    ...
  } else if (index_size == sizeof(u1)) {
    load_unsigned_byte(index, Address(r13, bcp_offset));
  } 
}
```

再看一下**ConstantPoolCacheEntry**定义，b1字段在方法完成解析后，即保存了字节码定义值(此处为0xb8)，若为解析则应该为0，所以可以通过简单的`__ cmpl(temp, (int) bytecode())`来判断是否完成解析。我们还需要再看一下路径2才能完全明白方法解析内涵。

```
// A ConstantPoolCacheEntry describes an individual entry of the constant
// pool cache. There's 2 principal kinds of entries: field entries for in-
// stance & static field access, and method entries for invokes. Some of
// the entry layout is shared and looks as follows:
//
// bit number |31                0|
// bit length |-8--|-8--|---16----|
// --------------------------------
// _indices   [ b2 | b1 |  index  ]  index = constant_pool_index
// _f1        [  entry specific   ]  metadata ptr (method or klass)
// _f2        [  entry specific   ]  vtable or res_ref index, or vfinal method ptr
// _flags     [tos|0|F=1|0|0|0|f|v|0 |0000|field_index] (for field entries)
// bit length [ 4 |1| 1 |1|1|1|1|1|1 |-4--|----16-----]
// _flags     [tos|0|F=0|M|A|I|f|0|vf|0000|00000|psize] (for method entries)
// bit length [ 4 |1| 1 |1|1|1|1|1|1 |-4--|--8--|--8--]
```

回看**路径2: 方法解析**

```c++
entry = CAST_FROM_FN_PTR(address, InterpreterRuntime::resolve_invoke);
```

<u>hotspot/src/share/vm/interpreter/interpreterRuntime.cpp</u>

其实方法解析到底是什么，就是通过常量池信息，回溯其定义类及基类，最终找到该方法精确定义位置(比如要实现多态调用等)并返回其在jvm内部的Method*。这一部分内容就和类的加载和解析关联起来，不详述。不过在**resolve_invoke**返回前，做了一件重要的事情

```c++
  switch (info.call_kind()) {
  case CallInfo::direct_call:
    cache_entry(thread)->set_direct_call(
      bytecode,
      info.resolved_method());
    break;
  case CallInfo::vtable_call:
    cache_entry(thread)->set_vtable_call(
      bytecode,
      info.resolved_method(),
      info.vtable_index());
    break;
    ......
  default:  ShouldNotReachHere();
  }
```

<u>hotspot/src/share/vm/oops/cpCache.cpp</u>

```c++
void ConstantPoolCacheEntry::set_direct_or_vtable_call(Bytecodes::Code invoke_code,
                                                       methodHandle method,
                                                       int vtable_index) {
  switch (invoke_code) {
      ......
    case Bytecodes::_invokestatic:
      set_method_flags(as_TosState(method->result_type()),
                       ((is_vfinal()               ? 1 : 0) << is_vfinal_shift) |
                       ((method->is_final_method() ? 1 : 0) << is_final_shift),
                       method()->size_of_parameters());
      set_f1(method());
      byte_no = 1;
      break;
    default:
      ShouldNotReachHere();
      break;
  }

  if (byte_no == 1) {
    set_bytecode_1(invoke_code);
  } 
  ......
}

void set_f1(Metadata* f1)                            {
    Metadata* existing_f1 = (Metadata*)_f1; // read once
    assert(existing_f1 == NULL || existing_f1 == f1, "illegal field change");
    _f1 = f1;
  }

void ConstantPoolCacheEntry::set_bytecode_1(Bytecodes::Code code) {
  OrderAccess::release_store_ptr(&_indices, _indices | ((u_char)code << bytecode_1_shift));
}
```

再去看一下**ConstantPoolCacheEntry**注释，即明白**set_f1**完成了静态类型方法**Method***设置，**set_bytecode_1**在**b1**位置保存了字节码定义值。此后即不需要再次解析，从**ConstantPoolCache**返回即可。这也就是方法解析及调用机制的内涵。

## 2.关键源码与汇编比照

下图是源码汇编实现与最终编译结果关联比照：

![image](https://github.com/aristotle0x01/aristotle0x01.github.io/assets/2216435/42fe0e2e-8a76-420c-96de-b4fb517113b0)

## 3.测试代码

```java
public class InterpretStatic {
    public static void main(String[] args) {
        InterpretStatic.int_static_test_method();
    }

    public static int int_static_test_method() {
        int i = 1;
        return i;
    }
}
```

关键信息：

```
  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=1, args_size=1
         0: invokestatic  #2                  // Method int_static_test_method:()I
         3: pop
         4: return
  
  Constant pool:
   #1 = Methodref          #4.#15         //  java/lang/Object."<init>":()V
   #2 = Methodref          #3.#16         //  InterpretStatic.int_static_test_method:()I
   #3 = Class              #17            //  InterpretStatic
   #4 = Class              #18            //  java/lang/Object
   #5 = Utf8               <init>
   #6 = Utf8               ()V
   #7 = Utf8               Code
   #8 = Utf8               LineNumberTable
   #9 = Utf8               main
  #10 = Utf8               ([Ljava/lang/String;)V
  #11 = Utf8               int_static_test_method
  #12 = Utf8               ()I
  #13 = Utf8               SourceFile
  #14 = Utf8               InterpretStatic.java
  #15 = NameAndType        #5:#6          //  "<init>":()V
  #16 = NameAndType        #11:#12        //  int_static_test_method:()I
  #17 = Utf8               InterpretStatic
  #18 = Utf8               java/lang/Object
```

## 4.debug过程

```shell
gdb /var/shared/openjdk/build/linux-x86_64-normal-server-slowdebug/jdk/bin/java

// 第一轮：保留入口点内存地址0x00007fffe1043690
run -XX:+PrintInterpreter InterpretStatic | grep -E -A 20 'invokestatic|zerolocals'

// jdk/src/share/bin/java.c
// 首先打且仅打该断点
b java.c:472 // breakpoint 1

run InterpretStatic

// 查验信息
p 'TemplateInterpreter::_active_table'.table_for(vtos)[184]
p 'AbstractInterpreter::_entry_table'[0]  // zerolocals

// 再次断点
b *0x00007fffe1043690 // invokestatic vtos入口
// 查验信息
p (unsigned char)*($r13)
p (unsigned char)*($r13+1)
p (unsigned char)*($r13+2)
p (unsigned char)*($r13+3)
p (unsigned char)*($r13+4)

// 再次断点，即将进入解释器入口zerolocals
b *0x00007fffe1043950 // jmpq   *0x58(%rbx)
p ((Method*)($rbx))->name() // int_static_test_method
```

图示1:

![image](https://github.com/aristotle0x01/aristotle0x01.github.io/assets/2216435/33eb3944-418d-4379-9879-f9b4db1cb801)

图示2:

![image](https://github.com/aristotle0x01/aristotle0x01.github.io/assets/2216435/076b5b63-adec-46ec-b547-2d4453e381f8)
