---
layout: post
title:  "java对象成员变量的内存布局及类的解析"
date:   2023-12-06
catalog: true
tags:
    - jvm
    - java
    - memory
    - object
---

## 0.背景

看jvm源码的时候，不禁会想：

* java程序中打印对象堆内存地址，和debug jvm时观测到的一样吗，怎么取得这些值？
* 对象成员变量在内存中如何布局，遵从定义顺序吗，**Unsafe::objectFieldOffset**获取的是什么，又是怎么实现的呢？
* 上述实现跟类的加载解析是如何关联的？
* 有类继承关系，有静态变量是会如何？

源码及环境参考：**jdk8u**，64位系统

## 1.打印对象内存布局[^4] [^5]

先来点感性认识，已经有很好的工具实现了这个功能：**Java Object Layout (JOL)**[^1]

### 1.1 测试对象类

```java
class SimpleInt {
    private int state;
}

class SimpleLong extends SimpleInt {
    private char state;
}

class FieldsArrangement extends SimpleLong {
    private boolean first;
    private char second;
    private double third;
    private int fourth;
    private boolean fifth;
    private static int si;
}

public static Unsafe getUnsafe() throws NoSuchFieldException, IllegalAccessException {
        Field f = Unsafe.class.getDeclaredField("theUnsafe"); //Internal reference
        f.setAccessible(true);
        return (Unsafe) f.get(null);
    }
```

### 1.2 测试类

```java
// -XX:-UseCompressedOops 为避免不必要麻烦，后面测试代码类同
public static void main(String args[]) throws Exception {
        FieldsArrangement fa = new FieldsArrangement();
        System.out.println(ClassLayout.parseInstance(fa).toPrintable());
    }
```

输出结果：

```java
oop.FieldsArrangement object internals:
OFF  SZ      TYPE DESCRIPTION                VALUE
  0   8           (object header: mark)      0x0000000000000001 (non-biasable; age: 0)
  8   8           (object header: class)     0x0000000112224848
 16   4       int SimpleInt.state            0
 20   4           (alignment/padding gap)    
 24   2      char SimpleLong.state            
 26   6           (alignment/padding gap)    
 32   8    double FieldsArrangement.third    0.0
 40   4       int FieldsArrangement.fourth   0
 44   2      char FieldsArrangement.second    
 46   1   boolean FieldsArrangement.first    false
 47   1   boolean FieldsArrangement.fifth    false
Instance size: 48 bytes
Space losses: 10 bytes internal + 0 bytes external = 10 bytes total
```

一些推断：

* 基类变量定义在前，如果不满8字节，会有填充；独立排序，以8字节(或者倍数)为界
* 静态变量理所应当不在对象内存里
* 变量内存顺序与定义顺序不同，且可能会有填充

### 1.3 对象内存布局

类是对象的模版，对象其实就是成员变量的结构体实例(当然也包含了类的指针)。jvm对象的内存布局，除了对象头里的 mark/kclass字段，主要就是成员变量。其布局主要考虑两点：

* 尽量节省空间
* 填充对齐以加速内存访问(4字节/8字节为准)

### 1.4 根据offset操作变量值

测试方法：

```java
public static void main(String args[]) throws Exception {
        Unsafe unsafe = getUnsafe();
        FieldsArrangement fa = new FieldsArrangement();
        Field f3 = FieldsArrangement.class.getDeclaredField("third");
        f3.setAccessible(true);
        assert unsafe.objectFieldOffset(f3) == 32;
        unsafe.putDouble(fa, 32, 2.0);

        System.out.println("field third: " + f3.get(fa));
    }
```

根据前面变量布局打印结果，可知**FieldsArrangement.third**偏移量为**32**，我们可以使用**Unsafe**来获取变量内存偏移量并且直接设置验证。

## 2.对象内存地址[^2] [^3]

测试方法：

```java
public static void main(String args[]) throws Exception {
        FieldsArrangement fa = new FieldsArrangement();
        System.out.println("object address: 0x" + Long.toHexString(VM.current().addressOf(fa)));
        System.out.println("object hashcode: 0x" + Long.toHexString(fa.hashCode()));
    }
```

输出 ：

```
object address: 0x174199a20
object hashcode: 0x1f89ab83
```

后续会研究验证**VM.current().addressOf**是如何实现的。



## 3.Unsafe.objectFieldOffset内部实现

为什么要看实现，将输出的表象与jvm的源码逻辑进行关联并作为学习源码特定功能的突破口。

### 3.1 定义溯源

方法定义：<u>jdk8u/jdk/src/share/classes/sun/misc/Unsafe.java</u>

​	**public native long objectFieldOffset(Field f);**

hotspot内部定义：

​	<u>jdk8u/hotspot/src/share/vm/prims/unsafe.cpp</u>

```c++
UNSAFE_ENTRY(jlong, Unsafe_ObjectFieldOffset(JNIEnv *env, jobject unsafe, jobject field))
  UnsafeWrapper("Unsafe_ObjectFieldOffset");
  return find_field_offset(field, 0, THREAD);
UNSAFE_END
```

那么关键看**find_field_offset**干了什么

### 3.2 实现逻辑

<u>jdk8u/hotspot/src/share/vm/prims/unsafe.cpp</u>

```c++
jint find_field_offset(jobject field, int must_be_static, TRAPS) {
  oop reflected   = JNIHandles::resolve_non_null(field);
  oop mirror      = java_lang_reflect_Field::clazz(reflected);
  Klass* k      = java_lang_Class::as_Klass(mirror);
  int slot        = java_lang_reflect_Field::slot(reflected);
  int modifiers   = java_lang_reflect_Field::modifiers(reflected);
  ......
  int offset = InstanceKlass::cast(k)->field_offset(slot);
  return field_offset_from_byte_offset(offset);
}
```

* **slot**: java类定义顺序

  | java.lang.reflect.Field | <img src="https://github.com/aristotle0x01/aristotle0x01.github.io/assets/2216435/32f70cc1-3b4c-453b-bb91-97267771f70e" alt="field" style="zoom:50%; float: left;" /> |
  | ----------------------- | ------------------------------------------------------------ |
  | Field debug信息         | <img src="https://github.com/aristotle0x01/aristotle0x01.github.io/assets/2216435/f14321e4-92a4-40ce-a895-c74d5174d8cc" alt="field debug" style="zoom:45%; float: left;" /> |

  * **find_field_offset**第一个参数**jobject field**即为**java.lang.reflect.Field**的一个实例
  * **java_lang_reflect_Field**为jvm中**java.lang.reflect.Field**的镜像类
  * **java_lang_reflect_Field::slot(reflected)**即根据类模版从对象中取出**slot**字段值，**modifiers,clazz**同理
  * 上述例子中**third**字段在类定义中排序为**2**，从0开始

* **offset**：对象内存偏移量 

关键就在`InstanceKlass::cast(k)->field_offset(slot)`，根据类定义，返回布局偏移量：

<u>jdk8u/hotspot/src/share/vm/oops/instanceKlass.hpp</u>

```c++
 private:
  FieldInfo* field(int index) const { return FieldInfo::from_field_array(_fields, index); }
 public:
  int     field_offset      (int index) const { return field(index)->offset(); }
```

<u>jdk8u/hotspot/src/share/vm/oops/fieldInfo.hpp</u>

```c++
static FieldInfo* from_field_array(Array<u2>* fields, int index) {
    return ((FieldInfo*)fields->adr_at(index * field_slots));
  }
```

可见最终返回的是**FieldInfo::offset()**，里面到底干了什么呢？

```c++
u4 offset() const {
    u2 lo = _shorts[low_packed_offset];
    switch(lo & FIELDINFO_TAG_MASK) {
      case FIELDINFO_TAG_OFFSET:
        return build_int_from_shorts(_shorts[low_packed_offset], _shorts[high_packed_offset]) >> FIELDINFO_TAG_SIZE;
      ......
    }
    ShouldNotReachHere();
    return 0;
  }

inline int build_int_from_shorts( jushort low, jushort high ) {
  return ((int)((unsigned int)high << 16) | (unsigned int)low);
}

#define FIELDINFO_TAG_SIZE             2
```

上述代码将**low_packed_offset, high_packed_offset**组合最终返回32位**offset**。至此，我们遍历了获取对象变量内存偏移量的过程。那么对象成员变量的这些布局信息是什么时候，由谁类决定的呢 ？

### 3.3 类成员变量定义信息

上面有两个关键信息**InstanceKlass**及其内部成员**_fields**。我们知道**InstanceKlass**是jvm中java类映射，每加载一个类，就会在jvm中生成一个**InstanceKlass**类对象，类本身也是一个特殊对象。

```c++
// Instance and static variable information, starts with 6-tuples of shorts
  // [access, name index, sig index, initval index, low_offset, high_offset]
  // for all fields, followed by the generic signature data at the end of
  // the array. Only fields with generic signature attributes have the generic
  // signature data set in the array. The fields array looks like following:
  //
  // f1: [access, name index, sig index, initial value index, low_offset, high_offset]
  // f2: [access, name index, sig index, initial value index, low_offset, high_offset]
  //      ...
  // fn: [access, name index, sig index, initial value index, low_offset, high_offset]
  //     [generic signature index]
  //     [generic signature index]
  //     ...
  Array<u2>*      _fields;
```

上面定义大体意思是，对于每一个成员变量，完成加载后其所有描述信息会形成一个**short[6]**长度的数组。当然所有变量描述(比如name index是常量池索引)数组是预定义了总体长度的，6元short数组是逻辑上的。从这里，也就明白了为什么前面说**slot**是变量定义顺序，并且细看前面**from_field_array**实现，slot是要乘以**field_slots**(乘以6)的。

那么上述信息是在什么时候生成的呢？在类加载的时候，并且做了变量布局的优化排序。

### 3.4 类成员变量的加载及布局

开门见山，自定义类加载器时，我们都知道有个**defineclass1**方法，其内部实现了**.class**文件的加载解析，构造常量池，成员变量，方法解析等。可以推测，肯定有个类似parser的类来做这个事情：

<u>jdk8u/hotspot/src/share/vm/classfile/classFileParser.hpp</u>

```c++
instanceKlassHandle parseClassFile(Symbol* name,
                                     ClassLoaderData* loader_data,
                                     Handle protection_domain,
                                     KlassHandle host_klass,
                                     GrowableArray<Handle>* cp_patches,
                                     TempNewSymbol& parsed_name,
                                     bool verify,
                                     TRAPS)
```

该方法实现极其庞大，仅截取相关部分：

```c++
  ClassFileStream* cfs = stream();
  u4 magic = cfs->get_u4_fast();

  // Version numbers
  u2 minor_version = cfs->get_u2_fast();
  u2 major_version = cfs->get_u2_fast();
  
  // Constant pool
  constantPoolHandle cp = parse_constant_pool(CHECK_(nullHandle));
  int cp_size = cp->length();
  
  // Access flags
  AccessFlags access_flags;
  jint flags = cfs->get_u2_fast() & JVM_RECOGNIZED_CLASS_MODIFIERS;
  
    u2 java_fields_count = 0;
    // Fields (offsets are filled in later)
    FieldAllocationCount fac;
    Array<u2>* fields = parse_fields(class_name,
                                     access_flags.is_interface(),
                                     &fac, &java_fields_count,
                                     CHECK_(nullHandle));
    // Methods
    bool has_final_method = false;
    AccessFlags promoted_flags;
    promoted_flags.set_flags(0);
    Array<Method*>* methods = parse_methods(access_flags.is_interface(),
                                            &promoted_flags,
                                            &has_final_method,
                                            &declares_default_methods,
                                            CHECK_(nullHandle));
 
  // sort methods
  intArray* method_ordering = sort_methods(methods);

  FieldLayoutInfo info;
  layout_fields(class_loader, &fac, &parsed_annotations, &info, CHECK_NULL);
    ......
  return this_klass;
}
```

* **parse_fields**实现了**.class**文件中成员变量的解析

  ```c++
  Array<u2>* ClassFileParser::parse_fields(Symbol* class_name,
                                           bool is_interface,
                                           FieldAllocationCount *fac,
                                           u2* java_fields_count_ptr, TRAPS) {
    ClassFileStream* cfs = stream();
    cfs->guarantee_more(2, CHECK_NULL);  // length
    u2 length = cfs->get_u2_fast();
    *java_fields_count_ptr = length;
  
    int num_injected = 0;
    InjectedField* injected = JavaClasses::get_injected(class_name, &num_injected);
    int total_fields = length + num_injected;
  
    // The field array starts with tuples of shorts
    // [access, name index, sig index, initial value index, byte offset].
    //
    //   f1: [access, name index, sig index, initial value index, low_offset, high_offset]
    //   f2: [access, name index, sig index, initial value index, low_offset, high_offset]
    //       ...
    ResourceMark rm(THREAD);
    u2* fa = NEW_RESOURCE_ARRAY_IN_THREAD(
               THREAD, u2, total_fields * (FieldInfo::field_slots + 1));
  
    // The generic signature slots start after all other fields' data.
    int generic_signature_slot = total_fields * FieldInfo::field_slots;
    int num_generic_signature = 0;
    for (int n = 0; n < length; n++) {
      cfs->guarantee_more(8, CHECK_NULL);  // access_flags, name_index, descriptor_index, attributes_count
  
      AccessFlags access_flags;
      jint flags = cfs->get_u2_fast() & JVM_RECOGNIZED_FIELD_MODIFIERS;
      verify_legal_field_modifiers(flags, is_interface, CHECK_NULL);
      access_flags.set_flags(flags);
  
      u2 name_index = cfs->get_u2_fast();
      int cp_size = _cp->length();
      check_property(valid_symbol_at(name_index),
        "Invalid constant pool index %u for field name in class file %s",
        name_index,
        CHECK_NULL);
      Symbol*  name = _cp->symbol_at(name_index);
      verify_legal_field_name(name, CHECK_NULL);
  
      u2 signature_index = cfs->get_u2_fast();
      check_property(valid_symbol_at(signature_index),
        "Invalid constant pool index %u for field signature in class file %s",
        signature_index, CHECK_NULL);
      Symbol*  sig = _cp->symbol_at(signature_index);
      verify_legal_field_signature(name, sig, CHECK_NULL);
  
      u2 constantvalue_index = 0;
      bool is_synthetic = false;
      u2 generic_signature_index = 0;
      bool is_static = access_flags.is_static();
      FieldAnnotationCollector parsed_annotations(_loader_data);
  
      FieldInfo* field = FieldInfo::from_field_array(fa, n);
      field->initialize(access_flags.as_short(),
                        name_index,
                        signature_index,
                        constantvalue_index);
      // Remember how many oops we encountered and compute allocation type
      FieldAllocationType atype = fac->update(is_static, type);
      field->set_allocation_type(atype);
    }
  
    int index = length;
    Array<u2>* fields = MetadataFactory::new_array<u2>(
            _loader_data, index * FieldInfo::field_slots + num_generic_signature,
            CHECK_NULL);
    _fields = fields; // save in case of error
    {
      int i = 0;
      for (; i < index * FieldInfo::field_slots; i++) {
        fields->at_put(i, fa[i]);
      }
      for (int j = total_fields * FieldInfo::field_slots;
           j < generic_signature_slot; j++) {
        fields->at_put(i++, fa[j]);
      }
      assert(i == fields->length(), "");
    }
  
    return fields;
  }
  
  ```

  * fields是变量数乘以FieldInfo::field_slots (6)加其它长度数组
  * 以类定义顺序完成成员变量的解析，但只处理了**access_flags，name index** 等变量 ，但没有处理 **low_packed_offset**与**high_packed_offse**
  * 再次说明**slot**是成员定义顺序

* **layout_fields**实现了对象布局的排序

  ```c++
  void ClassFileParser::layout_fields(Handle class_loader,...) {
    // Iterate over fields again and compute correct offsets.
    for (AllFieldStream fs(_fields, _cp); !fs.done(); fs.next()) {
      // skip already laid out fields
      if (fs.is_offset_set()) continue;
  
      // contended instance fields are handled below
      if (fs.is_contended() && !fs.access_flags().is_static()) continue;
  
      int real_offset = 0;
      FieldAllocationType atype = (FieldAllocationType) fs.allocation_type();
  
      // pack the rest of the fields
      switch (atype) {
        case STATIC_OOP:
          real_offset = next_static_oop_offset;
          next_static_oop_offset += heapOopSize;
          break;
        case STATIC_BYTE:
        	......
          break;
        case NONSTATIC_OOP:
        	......
          break;
        case NONSTATIC_BYTE:
          if( nonstatic_byte_space_count > 0 ) {
            real_offset = nonstatic_byte_space_offset;
            nonstatic_byte_space_offset += 1;
            nonstatic_byte_space_count  -= 1;
          } else {
            real_offset = next_nonstatic_byte_offset;
            next_nonstatic_byte_offset += 1;
          }
          break;
        case NONSTATIC_SHORT:
        	......
          break;
        default:
          ShouldNotReachHere();
      }
      fs.set_offset(real_offset);
    }
  }
  ```

  **fs.set_offset(real_offset)**最终又回到： <u>jdk8u/hotspot/src/share/vm/oops/fieldInfo.hpp</u>

  ```c++
  void set_offset(u4 val)                        {
      val = val << FIELDINFO_TAG_SIZE; // make room for tag
      _shorts[low_packed_offset] = extract_low_short_from_int(val) | FIELDINFO_TAG_OFFSET;
      _shorts[high_packed_offset] = extract_high_short_from_int(val);
    }
  ```

  至此完成计算并设置了某个变量最终在对象布局中的最终偏移量。

## 4.VM.current().addressOf())是如何取内存地址的

### 4.1 实现

<u>org/openjdk/jol/vm/HotspotUnsafe.java</u>：

```java
arrayObjectBase = U.arrayBaseOffset(Object[].class); // 初始化，数组中第一个元素偏移量

public long addressOf(Object o) {
        Object[] array = BUFFERS.get();

        array[0] = o;

        long objectAddress;
        switch (oopSize) {
            case 4:
                // 返回数组中偏移量处取值
                objectAddress = U.getInt(array, arrayObjectBase) & 0xFFFFFFFFL;
                break;
            case 8:
                objectAddress = U.getLong(array, arrayObjectBase);
                break;
            default:
                throw new Error("unsupported address size: " + oopSize);
        }

        array[0] = null;

        return toNativeAddress(objectAddress);
    }
```

对象数组第一个元素值，即为对象引用值，也即其内存地址

### 4.2 jvm debug验证

使用**VM.current().addressOf()**来获得对象内存地址，同时在jvm中debug该段代码，检查jvm生成对象后的返回地址，做交叉验证。

#### 4.2.1 验证代码

```java
// AddressTest.java
public static void main(String args[]) throws Exception {
        FieldsArrangement2 fa = new FieldsArrangement2();
        System.out.println("object address:");
        System.out.println("    0x" + Long.toHexString(VM.current().addressOf(fa)));
    }
```

#### 4.2.2 编译执行

```shell
../javac -cp ".:jol-core-0.17.jar" AddressTest.java
../java -cp ".:jol-core-0.17.jar" -XX:-UseCompressedOops AddressTest
gdbserver  :1234 ../java -cp ".:jol-core-0.17.jar" -XX:-UseCompressedOops AddressTest
```

#### 4.2.3 jvm debug步骤[^6]

* jdk8u/hotspot/src/share/vm/prims/jvm.cpp:1197 增加断点

  `condition 1 $_streq(str, "FieldsArrangement2")`，类加载后即将开始生成对象，所以在这个地方断点

* 到达上述断点后，打第二个断点，jdk8u/hotspot/src/share/vm/oops/instanceKlass.cpp:1161，即下面第13行

  在debug console `p this->external_name()`输出**"FieldsArrangement2"**，确认正在生成FieldsArrangement2对象

  ```c++
  instanceOop InstanceKlass::allocate_instance(TRAPS) {
    bool has_finalizer_flag = has_finalizer(); // Query before possible GC
    int size = size_helper();  // Query before forming handle.
  
    KlassHandle h_k(THREAD, this);
  
    instanceOop i;
  
    i = (instanceOop)CollectedHeap::obj_allocate(h_k, size, CHECK_NULL);
    if (has_finalizer_flag && !RegisterFinalizersAtInit) {
      i = register_finalizer(i, CHECK_NULL);
    }
    return i;
  }
  ```

* 打印输出FieldsArrangement2对象地址：`p i`

结果如下：

| jvm debug输出地址 | <img src="https://github.com/aristotle0x01/aristotle0x01.github.io/assets/2216435/8ae0d770-d17d-4274-a3ba-7da470a08ff0" alt="new instance" style="zoom:40%; float: left;" /> |
| ----------------- | ------------------------------------------------------------ |
| java程序输出地址  | <img src="https://github.com/aristotle0x01/aristotle0x01.github.io/assets/2216435/ce26c927-bbb1-416f-8835-a204c3d68b17" alt="java addressof" style="zoom:50%; float: left;" /> |

#### 4.2.4 java.lang.class与 InstanceKlass的关系(hotspot)

```java
Class<?> t = AddressTest.class;
        System.out.println("java clazz address: 0x" + Long.toHexString(VM.current().addressOf(AddressTest.class)));
        System.out.println("                    0x" + Long.toHexString(VM.current().addressOf(t)));
```

java.lang.class是jdk中对java类的描述，而InstanceKlass是jvm中对java类的表达，二者本质上表达了同一个东西，又略有不同。实际上InstanceKlass基类Klass内部包含了指针**oop  _java_mirror**，指向java.lang.class对象。可以类似debug查验指针内容和java输出：

```
// jvm debug
p _klass->java_mirror()
	$5 = (oopDesc *) 0x7fffcc874ff8
p *(_klass->java_mirror())
	$6 = {_mark = 0x5, _metadata = {_klass = 0x7fffa1157f00, _compressed_klass = 2702540544}, static _bs = 0x7ffff001b730}

// java output
java clazz address: 0x7fffcc874ff8
```

可见，**_java_mirror**确实建立了二者之间的关联

## 5.references

[^1]: [Java Object Layout (JOL)](https://github.com/openjdk/jol)
[^2]: [Memory address of variables in Java](https://stackoverflow.com/questions/1961146/memory-address-of-variables-in-java)
[^3]: [How can I get the memory location of a object in java?](https://stackoverflow.com/questions/7060215/how-can-i-get-the-memory-location-of-a-object-in-java)
[^4]: [Memory Layout of Objects in Java ](https://www.baeldung.com/java-memory-layout)
[^5]: [Java Objects Inside Out ](https://shipilev.net/jvm/objects-inside-out/)
[^6]: [centos7_build_openjdk8](https://github.com/aristotle0x01/centos7_build_openjdk8)
