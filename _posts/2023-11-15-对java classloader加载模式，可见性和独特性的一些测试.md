---
layout: post
title:  "对java classloader加载模式，可见性和独特性的一些测试"
date:   2023-11-15
categories: java classloader
---



## 0.why？

java **classloader**这个topic可以说是个java boy都要唠两句，烂大街了。这次打算细致`深入`研究jvm的时候，发现**<u>每个类加载器都有自己的namespace</u>**这句话给我带来了一些困扰，**namespace**在哪里？对delegation, visibility 和 uniqueness这三个重要特性有何影响？

<br/>

## 1.溯源加载

* 确保类型安全
* 类型隔离，多次或者反复加载

| delegate model                                               |
| :----------------------------------------------------------- |
| <img src="https://user-images.githubusercontent.com/2216435/283042893-f29e69bd-b536-425e-ba48-85e190548417.png" alt="delegation model" style="zoom:100%; float: left;" /> |

<br/>

### 1.1 java.lang.ClassLoader

<details>
<summary>**核心方法->**</summary>

{% highlight java %}

public Class<?> loadClass(String name) throws ClassNotFoundException {
        return loadClass(name, false);
    }

protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {}

​         if (c == null) {
​                    // If still not found, then invoke findClass in order
​                    // to find the class.
​                    long t1 = System.nanoTime();
​                    c = findClass(name);
​                }

​            }
​            if (resolve) {
​                resolveClass(c);
​            }
​            return c;

​        }

​    }

protected Class<?> findClass(String name) throws ClassNotFoundException {
        throw new ClassNotFoundException(name);
    }

{% endhighlight %}

</details>

* loadClass (public)是一般情况下对外提供的API，具体实现在loadClass (protected)
* loadClass (protected)
  * 首先确认是否已经加载，若是则直接返回缓存
  * 尝试从上层加载，符合层级加载模式
  * 否则从**findClass**加载
* **findClass**是留给自定义实现的入口

<br/>

### 1.2 AppClassLoader

jdk默认的应用类加载器：**jdk/src/share/classes/sun/misc/Launcher$AppClassLoader**

```java
public Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException
        {	
        		...
            if (ucp.knownToNotExist(name)) {
                // The class of the given name is not found in the parent
                // class loader as well as its local URLClassPath.
                // Check if this class has already been defined dynamically;
                // if so, return the loaded class; otherwise, skip the parent
                // delegation and findClass.
                Class<?> c = findLoadedClass(name);
                if (c != null) {
                    if (resolve) {
                        resolveClass(c);
                    }
                    return c;
                }
                throw new ClassNotFoundException(name);
            }

            return (super.loadClass(name, resolve));
        }
```

基类**URLClassLoader**实现了**findClass** (->**native defineClass1**)，从源文件读取二进制内容，此处不表。

核心逻辑：

* findLoadedClass尝试从缓存返回
* 否则从父层级递归加载

接下来略看一下<span style="color:blue">findLoadedClass--></span><span style="color:red">findLoadedClass0</span> 和<span style="color:blue">findClass--></span><span style="color:red">defineClass1</span>在jvm内部的实现。

<br/>

### 1.3 findLoadedClass0与defineClass1

#### 1.3.1 **findLoadedClass0**

**jdk/src/share/native/java/lang/ClassLoader.c**

```c++
	Java_java_lang_ClassLoader_findLoadedClass0(JNIEnv *env, jobject loader,
                                             jstring name)
  {
      if (name == NULL) {
          return 0;
      } else {
          return JVM_FindLoadedClass(env, loader, name);
      }
  }
```

**hotspot/src/share/vm/prims/jvm.cpp**

```c++
JVM_ENTRY(jclass, JVM_FindLoadedClass(JNIEnv *env, jobject loader, jstring name))
  ...
  Klass* k = SystemDictionary::find_instance_or_array_klass(klass_name,
                                                              h_loader,
                                                              Handle(),
                                                              CHECK_NULL);
#if INCLUDE_CDS
  if (k == NULL) {
    // If the class is not already loaded, try to see if it's in the shared
    // archive for the current classloader (h_loader).
    instanceKlassHandle ik = SystemDictionaryShared::find_or_load_shared_class(
        klass_name, h_loader, CHECK_NULL);
    k = ik();
  }
#endif
  return (k == NULL) ? NULL :
            (jclass) JNIHandles::make_local(env, k->java_mirror());
JVM_END
```

**SystemDictionary**是一个类似**hashmap**数据结构，可见其核心逻辑即为根据class name从中取出类对象。

<br/>

#### 1.3.2 **defineClass1**

```c++
JNIEXPORT jclass JNICALL
Java_java_lang_ClassLoader_defineClass1(JNIEnv *env,
                                        jobject loader,
                                        jstring name,
                                        jbyteArray data,
                                        ...)
{
    ...
    result = JVM_DefineClassWithSource(env, utfName, loader, body, length, pd, utfSource);

    if (utfSource && utfSource != sourceBuf)
        free(utfSource);
    ...
}
```

**jvm_define_class_common**:

```c++
// common code for JVM_DefineClass() and JVM_DefineClassWithSource()
// and JVM_DefineClassWithSourceCond()
static jclass jvm_define_class_common(JNIEnv *env, const char *name,
                                      jobject loader, const jbyte *buf,
                                      jsize len, jobject pd, const char *source,
                                      jboolean verify, TRAPS) {
  ...
  Handle protection_domain (THREAD, JNIHandles::resolve(pd));
  Klass* k = SystemDictionary::resolve_from_stream(class_name, class_loader,
                                                     protection_domain, &st,
                                                     verify != 0,
                                                     CHECK_NULL);
	...
  return (jclass) JNIHandles::make_local(env, k->java_mirror());
}
```

**SystemDictionary::resolve_from_stream**: 完成文件读取解析，并放入hashmap

```c++
// Add a klass to the system from a stream (called by jni_DefineClass and
// JVM_DefineClass).
Klass* SystemDictionary::resolve_from_stream(Symbol* class_name,
                                             Handle class_loader,
                                             Handle protection_domain,
                                             ClassFileStream* st,
                                             bool verify,
                                             TRAPS) {
  ...
  ClassFileParser parser(st);
  instanceKlassHandle k = parser.parseClassFile(class_name,
                                                loader_data,
                                                protection_domain,
                                                parsed_name,
                                                verify,
                                                THREAD);
  	...
    // Add class just loaded
    if (is_parallelCapable(class_loader)) {
      k = find_or_define_instance_class(class_name, class_loader, k, THREAD);
    } else {
      define_instance_class(k, THREAD);
    }
  }

  return k();
}
```

**define_instance_class**: 放入hashmap

```c++
void SystemDictionary::define_instance_class(instanceKlassHandle k, TRAPS) {
  ...
  // Add the new class. We need recompile lock during update of CHA.
  {
    unsigned int p_hash = placeholders()->compute_hash(name_h, loader_data);
    int p_index = placeholders()->hash_to_index(p_hash);

    MutexLocker mu_r(Compile_lock, THREAD);

    // Add to class hierarchy, initialize vtables, and do possible
    // deoptimizations.
    add_to_hierarchy(k, CHECK); // No exception, but can block

    // Add to systemDictionary - so other classes can see it.
    // Grabs and releases SystemDictionary_lock
    update_dictionary(d_index, d_hash, p_index, p_hash,
                      k, class_loader_h, THREAD);
  }
  k->eager_initialize(THREAD);
  ...
  post_class_define_event(k(), loader_data);
}
```

<br/>

## 2.一些测试

自定义类加载器**CustomClassLoader**

```java
static class CustomClassLoader extends ClassLoader {
    @Override
    public Class findClass(String name) throws ClassNotFoundException {
        byte[] b = loadClassFromFile(name);
        return defineClass(name, b, 0, b.length);
    }

    private byte[] loadClassFromFile(String fileName)  {
        InputStream inputStream = getClass().getClassLoader().getResourceAsStream(
                fileName.replace('.', File.separatorChar) + ".class");
        byte[] buffer;
        ByteArrayOutputStream byteStream = new ByteArrayOutputStream();
        int nextValue = 0;
        try {
            while ( (nextValue = inputStream.read()) != -1 ) {
                byteStream.write(nextValue);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        buffer = byteStream.toByteArray();
        return buffer;
    }
}
```

测试加载类

```java
public class Test1 {
    public int a1;

    @Override
    public boolean equals(Object obj) {
        if (obj == null) return false;
        if (getClass() != obj.getClass())
            return false;
        return a1 == (((Test1)(obj)).a1);
    }

    @Override
    public int hashCode() {
        return a1;
    }
}
```

<br/>

### 2.1 loadClass vs findClass

* findclass 会新生成类对象，故加载器为**CustomClassLoader**； loadClass使用默认逻辑，类加载器为**AppClassLoader**
* **classl**与**classf**为不同类对象
* **classl**与**classn**完全相同(第二次从缓存加载)

```java
{
	CustomClassLoader classLoader1 = new CustomClassLoader();
	Class<?> classl = classLoader1.loadClass("classloader.Test1");
	Class<?> classf = classLoader1.findClass("classloader.Test1");
	Class<?> classn = Test1.class;

	System.out.println("findClass:" + classf.getClassLoader());
	System.out.println("loadClass:" + classl.getClassLoader());
	System.out.println("classn: " + classn.getClassLoader());
	System.out.println("classl==classf:" + (classl==classf));
	System.out.println("classl equals classf:" + (classl.equals(classf)));
	System.out.println("classf==classn:" + (classf==classn));
	System.out.println("classf equals classn:" + (classf.equals(classn)));
	// classl 与 classn是完全相同的一个类对象
	System.out.println("classl==classn:" + (classl==classn));
	System.out.println("classl equals classn:" + (classl.equals(classn)));
	System.out.println("");
}

// output
findClass:classloader.ClassLoaderTest$CustomClassLoader@6ed3ef1
loadClass:sun.misc.Launcher$AppClassLoader@18b4aac2
classn: sun.misc.Launcher$AppClassLoader@18b4aac2
classl==classf:false
classl equals classf:false
classf==classn:false
classf equals classn:false
classl==classn:true
classl equals classn:true
```

| classl == classn                                             |
| ------------------------------------------------------------ |
| <img src="https://user-images.githubusercontent.com/2216435/283103479-bd679e81-a6b7-4c2f-9126-149e471eeea7.png" alt="delegation model" style="zoom:80%; float: left;" /> |

<br/>

### 2.2 同一个加载器类

* 两个加载器实例(类相同)加载同一个类，结果不同

```java
{
	// 不同加载器，则类不同
	CustomClassLoader classLoader1 = new CustomClassLoader();
	Class<?> class1 = classLoader1.findClass("classloader.Test1");

	CustomClassLoader classLoader2 = new CustomClassLoader();
	Class<?> class2 = classLoader2.findClass("classloader.Test1");

	System.out.println("classLoader1:" + class1.getClassLoader());
	System.out.println("classLoader2:" + class2.getClassLoader());
	System.out.println("class1:" + class1 + "@" + Integer.toHexString(class1.hashCode()));
	System.out.println("class2:" + class2 + "@" + Integer.toHexString(class2.hashCode()));
	System.out.println("class1==class2:" + (class1==class2));
	System.out.println("class1 equals class2:" + (class1.equals(class2)));

	System.out.println("");
}

// output
classLoader1:classloader.ClassLoaderTest$CustomClassLoader@6ed3ef1
classLoader2:classloader.ClassLoaderTest$CustomClassLoader@e73f9ac
class1:class classloader.Test1@7b1d7fff
class2:class classloader.Test1@299a06ac
class1==class2:false
class1 equals class2:false
```

<br/>

### 2.3 不同类加载器加载类产生的对象

* 不同类加载器加载类产生对象，则对象不同；若类相同

* **注意[^1]：Test1: equals**重载：**`if (getClass() != obj.getClass())`**，类若由不同加载器加载，则**equals**方法为**false**;

  反过来说，**equals**重载时，也需要注意类是否同一个加载器加载

```java
{
	CustomClassLoader classLoader1 = new CustomClassLoader();
	Class<?> class1 = classLoader1.loadClass("classloader.Test1");
	Class<?> classf = classLoader1.findClass("classloader.Test1");

	Class<?> class2 = Test1.class;
	System.out.println("equals: " + (class1 == class2));

	Test1 t3 = new Test1();
	Test1 t4 = new Test1();
	System.out.println("t3==t4: " + (t3 == t4));
	System.out.println("t3 equals t4: " + (t3.equals(t4)));

	Object o1 = class1.newInstance();
	Field f1 = class1.getField("a1");
	f1.setInt(o1, 1);
	Test1 o2 = (Test1) class2.newInstance();
	Field f2 = class2.getField("a1");
	f2.setInt(o2, 1);
	System.out.println("o1 equals o2: " + (o2.equals(o1)));
	Object of = classf.newInstance();
	Field ff = classf.getField("a1");
	ff.setInt(of, 1);
  System.out.println("o2 equals of: " + (o2.equals(of)));
}

// output
equals: true
t3==t4: false
t3 equals t4: true
o1 equals o2: true
o2 equals of: false
```

<br/>

## 3. mysql驱动加载

### 3.1 加载方式

* java.sql.DriverManager通过系统属性jdbc.drivers加载(启动加jvm参数 -Djdbc.drivers=com.mysql.jdbc.Driver)
* 通过Class.forName("com.mysql.jdbc.Driver")加载
* java.sql.DriverManager调用getConnection时加载

本文为了测试类加载过程，选择第三种

<br/>

### 3.2 自行加载

**测试代码：**

```
String url = "jdbc:mysql://localhost:3306/mysql?serverTimezone=GMT%2B8";
String user = "root";
String password = "123456";
Connection connections = DriverManager.getConnection(url, user, password);
```

步骤1: `java/sql/DriverManager.java`

触发DriverManager类加载，并执行静态块

```
static {
        loadInitialDrivers();
        println("JDBC DriverManager initialized");
    }
```

**loadInitialDrivers**中经由`ServiceLoader.load(Driver.class)` 和`driversIterator.next()`完成`com.mysql.jdbc.Driver`类加载

```java
private static void loadInitialDrivers() {
				...
				
        AccessController.doPrivileged(new PrivilegedAction<Void>() {
            public Void run() {

                ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
                Iterator<Driver> driversIterator = loadedDrivers.iterator();

                /* Load these drivers, so that they can be instantiated.*/
                try{
                    while(driversIterator.hasNext()) {
                        driversIterator.next();
                    }
                } catch(Throwable t) {
                // Do nothing
                }
                return null;
            }
        });

        ...
    }
```

步骤2: `com/mysql/jdbc/Driver.java`

```java
static {
        try {
            java.sql.DriverManager.registerDriver(new Driver());
        } catch (SQLException E) {
            throw new RuntimeException("Can't register driver!");
        }
    }
```

步骤3: `DriverManager.getConnection`逐个尝试驱动，直到链接成功

<br/>

### 3.3 就这么简单吗？

从上面的步骤来看，加载过程似乎平淡无奇。其实有点内涵，**java.sql.DriverManager**由**bootstrapclassloader**加载(debug验证classloader时返回null)。如此说来loadInitialDrivers中`com.mysql.jdbc.Driver`也应当由**bootstrapclassloader**加载(caller规则)，但这显然是违反jdk安全的，而且经测试其实是由`AppClassLoader`(它才能扫描到应用CLASSPATH)加载的。问题出在哪里呢[^3]？

```java
public static <S> ServiceLoader<S> load(Class<S> service) {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return ServiceLoader.load(service, cl);
    }
```

`Thread.currentThread().getContextClassLoader()`一般情况下返回`AppClassLoader`。如此，也就实现了SPI类型加载安全机制。

* 实际上`AppClassLoader`仍然会回溯到**bootstrapclassloader**尝试加载，符合父加载模式
* **bootstrapclassloader**则已到顶端，既没有更上层加载器，自身也无法加载
* ContextClassLoader[^2]相当于主动降低了加载层级

<br/>

## 4.tomcat 类加载机制

| tomcat classloader hierarchy                                 |
| ------------------------------------------------------------ |
| <img src="https://user-images.githubusercontent.com/2216435/283347896-d144639f-47d4-4b3e-80c2-cee6d9debdee.png" alt="tomcat delegation model" style="zoom:100%; float: left;" /> |

tomcat加载规则，可见webapp层打破了delegation默认加载规则[^4]：

| Web application class loader | \$CATALINA_HOME/webapps/<webapp>/WEB-INF/lib<br/>\$CATALINA_HOME/webapps/<webapp>/WEB-INF/classes |
| ---------------------------- | ------------------------------------------------------------ |
| bootstrap class loader       | \$JAVA_HOME/jre/lib                                          |
| Extension class loader       | \$JAVA_HOME/jre/lib/ext                                      |
| system class loader          | \$CATALINA_HOME/bin/bootstrap.jar<br/>\$CATALINA_HOME/bin/tomcat-juli.jar |
| common class loader          | \$CATALINA_HOME/lib                                          |

<br/>

## 5.references

[^1]: [Why do I need to override the equals and hashCode methods in Java?](https://stackoverflow.com/questions/2265503/why-do-i-need-to-override-the-equals-and-hashcode-methods-in-java)
[^2]: [Difference between thread's context class loader and normal classloader](https://stackoverflow.com/questions/1771679/difference-between-threads-context-class-loader-and-normal-classloader)
[^3]:[为什么需要ContextClassLoader](https://www.cnblogs.com/guiblog/p/14244064.html)

[^4]: [Class Loaders ](http://www.datadisk.co.uk/html_docs/java_app/tomcat6/tomcat6_classloaders.htm)

