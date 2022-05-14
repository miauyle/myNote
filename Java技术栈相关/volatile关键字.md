# volatile关键字

## 作用：

#### 1.volatile变量的可见性。

JMM中规定所有的变量都存储在主内存（Main Memory）中，每条线程都有自己的工作内存（Work Memory），线程的工作内存中保存了该线程所使用的变量的从主内存中拷贝的副本。线程对于变量的读、写都必须在工作内存中进行，而不能直接读、写主内存中的变量。同时，本线程的工作内存的变量也无法被其他线程直接访问，必须通过主内存完成。

<div align="center">
    <img src="https://pic2.zhimg.com/80/v2-0b8d013f211b305e76aa052db3e59a7d_1440w.jpg" width="500px">


- 当对volatile变量执行写操作后，JMM会把工作内存中的最新变量值强制刷新到主内存
- 写操作会导致其他线程中的缓存无效

这样，其他线程使用缓存时，发现本地工作内存中此变量无效，便从主内存中获取，这样获取到的变量便是最新的值，实现了线程的可见性。

#### 2.volatile变量的有序性

通过禁止指令重排实现有序性，内存屏障来禁止指令重排

  > `volatile`是通过编译器在生成字节码时，在指令序列中添加“**内存屏障**”来禁止指令重排序的。
  >
  > JMM层面的“**内存屏障**”：
  >
  > - **LoadLoad屏障**： 对于这样的语句Load1; LoadLoad; Load2，在Load2及后续读取操作要读取的数据被访问前，保证Load1要读取的数据被读取完毕。
  > - **StoreStore屏障**：对于这样的语句Store1; StoreStore; Store2，在Store2及后续写入操作执行前，保证Store1的写入操作对其它处理器可见。
  > - **LoadStore屏障**：对于这样的语句Load1; LoadStore; Store2，在Store2及后续写入操作被刷出前，保证Load1要读取的数据被读取完毕。
  > - **StoreLoad屏障**： 对于这样的语句Store1; StoreLoad; Load2，在Load2及后续所有读取操作执行前，保证Store1的写入对所有处理器可见。
  >
  > JVM的实现会在volatile读写前后均加上内存屏障，在一定程度上保证有序性。如下所示：
  >
  > > LoadLoadBarrier
  > > volatile 读操作
  > > LoadStoreBarrier
  > >
  >> StoreStoreBarrier
  > > volatile 写操作
  >> StoreLoadBarrier

## 底层实现

#### 1.java代码

```java
public class TestVolatile {

    public static volatile int counter = 1;

    public static void main(String[] args){
        counter = 2;
        System.out.println(counter);
    }
}
```



#### 2.字节码

`javap -v TestVolatile.class`命令反编译查看字节码文件

```
{
  public static volatile int counter;
    descriptor: I
    flags: ACC_PUBLIC, ACC_STATIC, ACC_VOLATILE

  public com.ya.bo.TestVolatile();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 3: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/ya/bo/TestVolatile;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: iconst_2
         1: putstatic     #2                  // Field counter:I
         4: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
         7: getstatic     #2                  // Field counter:I
        10: invokevirtual #4                  // Method java/io/PrintStream.println:(I)V
        13: return
      LineNumberTable:
        line 8: 0
        line 9: 4
        line 10: 13
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      14     0  args   [Ljava/lang/String;

  static {};
    descriptor: ()V
    flags: ACC_STATIC
    Code:
      stack=1, locals=0, args_size=0
         0: iconst_1
         1: putstatic     #2                  // Field counter:I
         4: return
      LineNumberTable:
        line 5: 0
}

```

可以看到，修饰`counter`字段的public、static、volatile关键字，在字节码层面分别是以下访问标志： **ACC_PUBLIC, ACC_STATIC, ACC_VOLATILE**

`volatile`在字节码层面，就是使用访问标志：**ACC_VOLATILE**来表示，供后续操作此变量时判断访问标志是否为ACC_VOLATILE，来决定是否遵循volatile的语义处理。

#### 3.jvm源码

上面main方法编译后的字节码，有`putstatic`和`getstatic`指令（如果是非静态变量，则对应`putfield`和`getfield`指令）来操作`counter`字段。那么对于被`volatile`变量修饰的字段，是如何实现`volatile`语义的，看源码：

hotspot/src/share/vm/interpreter/bytecodeInterpreter.cpp文件中的 putstatic 指令代码

重点判断逻辑`cache->is_volatile()`方法，调用的是`/hotspot/src/share/vm/utilitie/accessFlags.hpp`文件中的方法，**用来判断访问标记是否为volatile修饰**。

下面一系列的if...else...对`tos_type`字段的判断处理，是针对java基本类型和引用类型的赋值处理。如：

```cpp
obj->release_byte_field_put(field_offset, STACK_INT(-1));
```

对byte类型的赋值处理，调用的是`/hotspot/src/share/vm/oops/oop.inline.hpp`文件中的方法：

```cpp
// load操作调用的方法
inline jbyte oopDesc::byte_field_acquire(int offset) const                  
{ return OrderAccess::load_acquire(byte_field_addr(offset));     }
// store操作调用的方法
inline void oopDesc::release_byte_field_put(int offset, jbyte contents)     
{ OrderAccess::release_store(byte_field_addr(offset), contents); }
```

赋值的操作又被包装了一层，又调用的**OrderAccess::release_store**方法。

OrderAccess是定义在`/hotspot/src/share/vm/runtime/orderAccess.hpp`头文件下的方法，具体的实现是根据不同的操作系统和不同的cpu架构，有不同的实现。

其实里面看它调用了c++的volatile，

c++的volatile作用

- volatile修饰的类型变量表示可以被某些编译器未知的因素更改（如：操作系统，硬件或者其他线程等）
- 使用 volatile 变量时，避免激进的优化。即：系统总是重新从内存读取数据，即使它前面的指令刚从内存中读取被缓存，防止出现未知更改和主内存中不一致

上面的if...else走完后，调用了**OrderAccess::storeload();** 即：只要volatile变量赋值完成后，都会走这段代码逻辑。

它依然是声明在`orderAccess.hpp`头文件中，在不同操作系统或cpu架构下有不同的实现。`orderAccess_bsd_x86.inline.hpp`是mac系统下x86架构的实现：

```c++
inline void OrderAccess::fence() {
  if (os::is_MP()) {
    // always use locked addl since mfence is sometimes expensive
#ifdef AMD64
    __asm__ volatile ("lock; addl $0,0(%%rsp)" : : : "cc", "memory");
#else
    __asm__ volatile ("lock; addl $0,0(%%esp)" : : : "cc", "memory");
#endif
  }
}
```

代码`lock; addl $0,0(%%rsp)` 其中的addl $0,0(%%rsp) 是把寄存器的值加0，相当于一个空操作

**lock前缀，会保证某个处理器对共享内存（一般是缓存行cacheline）的独占使用。它将本处理器缓存写入内存，为了保证各个处理器缓存值是一致的，会实现缓存一致性协议，每个处理器通过嗅探在总线上传播的数据检查自己缓存值是否过期，当处理器发现自己缓存行对应内存地址被修改，就会把当前处理器缓存行设置成无效状态，处理器对数据进行修改操作，从共享内存把数据读到处理器缓存里。**

**通过独占内存、使其他处理器缓存失效，达到了“指令重排序无法越过内存屏障”的作用**

#### 4.汇编指令中的lock的前缀指令

 lock addl $0x0,(%rsp)

```
0x000000010df14ccf: je     0x000000010df14d13
  0x000000010df14cd5: push   %rsi
  0x000000010df14cd6: push   %rdx
  0x000000010df14cd7: push   %rcx
  0x000000010df14cd8: push   %r8
  0x000000010df14cda: push   %r9
  0x000000010df14cdc: movabs $0x10b57a898,%rsi  ;   {metadata({method} {0x000000010b57a898} 'arraycopy' '(Ljava/lang/Object;ILjava/lang/Object;II)V' in 'java/lang/System')}
  0x000000010df14ce6: mov    %r15,%rdi
  0x000000010df14ce9: test   $0xf,%esp
  0x000000010df14cef: je     0x000000010df14d07
  0x000000010df14cf5: sub    $0x8,%rsp
  0x000000010df14cf9: callq  0x0000000109ef13f4  ;   {runtime_call}
  0x000000010df14cfe: add    $0x8,%rsp
  0x000000010df14d02: jmpq   0x000000010df14d0c
  0x000000010df14d07: callq  0x0000000109ef13f4  ;   {runtime_call}
  0x000000010df14d0c: pop    %r9
  0x000000010df14d0e: pop    %r8
  0x000000010df14d10: pop    %rcx
  0x000000010df14d11: pop    %rdx
  0x000000010df14d12: pop    %rsi
  0x000000010df14d13: lea    0x200(%r15),%rdi
  0x000000010df14d1a: movl   $0x4,0x278(%r15)
  0x000000010df14d25: callq  0x0000000109d393d6  ;   {runtime_call}
  0x000000010df14d2a: vzeroupper 
  0x000000010df14d2d: movl   $0x5,0x278(%r15)
  // lock指令
  0x000000010df14d38: lock addl $0x0,(%rsp)
  0x000000010df14d3d: cmpl   $0x0,-0x3deacf7(%rip)        # 0x000000010a12a050
                                                ;   {external_word}
  0x000000010df14d47: jne    0x000000010df14d5b
  0x000000010df14d4d: cmpl   $0x0,0x30(%r15)
  0x000000010df14d55: je     0x000000010df14d74
  0x000000010df14d5b: mov    %r15,%rdi
  0x000000010df14d5e: mov    %rsp,%r12
  0x000000010df14d61: sub    $0x0,%rsp
  0x000000010df14d65: and    $0xfffffffffffffff0,%rsp
  0x000000010df14d69: callq  0x0000000109f66bfa  ;   {runtime_call}
  0x000000010df14d6e: mov    %r12,%rsp
  0x000000010df14d71: xor    %r12,%r12
  0x000000010df14d74: movl   $0x8,0x278(%r15)
  0x000000010df14d7f: cmpl   $0x1,0x2a4(%r15)
  0x000000010df14d8a: je     0x000000010df14e12
  0x000000010df14d90: cmpb   $0x0,-0x3df5575(%rip)        # 0x000000010a11f822
                                                ;   {external_word}
```

