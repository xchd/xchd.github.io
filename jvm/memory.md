# 内存 #

虚拟机的通用实现是，用于存储指令和操作数的一个数据结构，用于函数调用操作的一个调用栈结构，一个指令指针，其指向下一条将要执行指令的指针，一个虚拟的CPU，负责如下的指令调度：取指令、解码操作数、执行指令，基于堆栈的虚拟机架构和基于寄存器的虚拟机架构
jvm是基于堆栈的虚拟机架构

在jvm中，内存可以划分为共有和私有两部分。私有的内存空间有虚拟机栈，本地方法栈和程序计算器；共有部分是堆和方法区；

#### 虚拟机栈 ####
先进后出,一个线程一个栈，每调用一个方法新增一个栈帧压栈；-Xss参数设置大小，1k大概可以调用10个方法；1M大概1万次调用。创建线程的多少取决于操作系统的内存大小；

栈帧：
栈帧包含了局部变量表，操作数栈，动态链接和方法出口等信息；
局部变量表的下标0是this，如果是对象存放的是地址。局部变量的赋值指令是istore，将int的值存入局部变量，会导致操作数栈出栈；

操作数栈常用指令iconst：将常量池的值压入操作数栈；iload：从局部变量装载int类型的值

#### 本地方法栈 ####
native方法，作用是与操作系统和外部环境交互；System.loadLibrary(Name) 对应共享库名libName.so 

#### 程序计算器 ####
记录当前执行的字节码指令的位置，执行引擎会修改该值，用于线程恢复

#### 方法区 ####
在hotspot中，jdk1.7 永久区，在堆中，jdk1.8 元素空间，在内存中
包含信息：类信息，常量和类静态变量。存储的信息是C++的信息:
Class clzz =xxx.getClass();clazz存在堆中不在方法区；

设置大小的参数：-XX:MetaspaceSize和-XX:MaxMetaspaceSize；如果不设置,默认21M,满了会触发GC,并进行动态的扩容或者缩容

#### 堆 ####
堆划分为新生代和老年代，新生代默认占1/3,老年代占2/3,新生代又划分为eden去和两个s区。默认占比是8:1:1;进入老年代的原因有大对象，长期存活对象（默认15次minor GC），动态年龄进入老年（年龄1,2,n的对象大于survivor 50%的时候,>=n的对象移到老年代），和老年代的空间担保（minor gc的时候s区装不下）；


配置参数是
-Xms memory size
-Xmx memory max
-XX:PermSizeMax
-XX:MaxMetaspaceSize

#### 堆外内存 ####
默认和堆大小一样，参数配置是XX:MaxDirectMemorySize，首先打开堆外内存跟踪，-XX:NativeMemoryTracking=detail，执行jcmp命令，jcmd pid VM.native_memory detail scale=MB > temp.txt  ；linux的pmap 查看进程内存地址空间，结合刚才的地址找出对应的线程；或者使用google-perftools

#### 配置 ####
首先进行估算；估算每秒生成多大的对象，持续多少秒之后新生代发生GC。（大小*频率*时间）合理安排大小，避免频繁GC；然后上测试环境，模拟压力测试，稳定性正常；一般情况是4核8G 并发500；

模板

    -Xms4096M -Xmx4096M -Xmn3072M -Xss1M 
    -XX:MetaspaceSize=256M -XX:MaxMetaspaceSize=256M 
    -XX:+UseParNewGC -XX:+UseConcMarkSweepGC 
    -XX:CMSInitiatingOccupancyFaction=92 
	-XX:+UseCMSCompactAtFullCollection 
    -XX:CMSFullGCsBeforeCompaction=0 
	-XX:+CMSParallelInitialMarkEnabled 
    -XX:+CMSScavengeBeforeRemark 
	-XX:+DisableExplicitGC -XX:+PrintGCDetails 
    -Xloggc:gc.log - XX:+HeapDumpOnOutOfMemoryError 
    -XX:HeapDumpPath=/usr/local/app/oom

