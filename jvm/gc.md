# GC #
GC收集器的三个考量指标：

占用的内存（Capacity）
延迟（Latency）
吞吐量（Throughput）

#### 触发和转换 ####
新生代内存不够用的时候就会触发GC；
老年代触发GC的条件有多个：
1在minorGC之前，检测老年代剩余内存是否小于新生代
2minorGC之后，转移到老年代的空间大于老年代的剩余内存
3老年代的空间不足
4方法区空间不足

minorGC执行之前，检查老年代的剩余空间是否能装下新生代的内容，如果可以，直接执行MinorGC。minorGC之后如果survivor不够装入存活的对象，部分就会被移到老年代，年龄超过15的也转移到老年代。年龄1,2,n占用survivor的50%,那么年龄>=n的对象也移到老年代(尽量保证GC之后小于survivor的50%)，如果老年代空间不够新生代的大且老年代空间大于从新生代进入老年代的平均大小，就执行minorGC，否则就执行majorGC.如果还是存不下就OOM

#### 垃圾分析 ####
垃圾分析算法可以使用计数器和可达性算法，jvm使用可达性算法实现。从GC Roots开始收集，局部变量，静态变量，常量，native引用变量都可以作为GC roots；加速了GC Roots的枚举，使用OopMap存放着对象引用的位置；在分析的引用关系的时候，引用的类型分为强引用，软应用和弱应用，虚引用。

分析元空间的类是否有效，满足以下约束：
所有实例被回收，该类类加载器被回收，该类的Class对象没有引用；
第二点很苛刻，应为jvm提供的三个类加载器（引导，扩展，应用）是不会被回收的。回收的只能是自定义的类加载器


#### 收集算法 ####
基于分代收集理论，算法有标记复制，标记清除，标记整理


#### 三色标记算法 ####
CMS和G1在垃圾对象扫描中，判断是否是垃圾，使用三色标记算法：
三色是白，灰，黑。
白色:未被收集器访问过.默认都是白色;分析完成白色的被清除
灰:被收集器访问过,但至少存在一个对象没有被扫描,分析未完成
黑:全部成员变量对象都被收集器扫描过,且不能回收.表示分析完成

需要解决错标的问题：
在并发收集的过程中，新产生的对象直接标记黑色，这个就产生了浮动垃圾。实现的手段是引用关系的变动使用c++的读写屏障实现，写前或者后做操作,记录信息等,叫写屏障，读前或者后做操作,记录信息等,叫读屏障。因为效率问题，读写屏障在特定阶段才会执行,比如GC回收阶段.在记录引用关系变动可以使用增量更新和原始快照（STAB）实现。增量更新是黑色指向白色对象,将该引用记录,等并发扫描结束后,将黑色对象作为根,重新扫描新增部分.原始快照是灰色对象要删除指向白色对象,记录删除的对象,扫描之后之后,将记录为根重新扫描,白色对象直接标记为黑色,让对象在本次GC存活(删除)；不同的收集器使用不同的方法。CMS使用写屏障+增量更新的方法。效率不如原始快照的快，但减少浮动垃圾；G1使用写屏障+原始快照，写的时候先写到队列中，然后操作队列生成原始快照，启动缓冲的作用。不需要重新标记阶段再次深度扫描被删除的引用.原始快照效率更高,但可能造成更多的浮动垃圾.白色部分标记为黑色。因为G1关注了暂停,不完全回收,所以多点浮动垃圾没关系.G1对象分布在不同的region区,如果使用增量更新,效率很低.所以只做简单标记


#### 夸代引用问题 ####
解决夸代引用问题可以使用记录老年代指向新生代的地址，或者是记录老年代引用新生代的对象，也可以是新生代的一段地址是否存在老年代的应用，也就是接下来要说的卡表。精度虽然比不上上面两种，但使用记录的内存更少了。
老年代引用了新生代，在新生代区域维护一个老年代引用新生代的记录集合remember set的数据结构，所以新生代回收GC Root + Remember Set；cardtable是remember set的一个实现,底层实现就是一个字节数组，老年代划分很多card页,卡页固定大小是512字节，如果老年代有引用新生代,在卡表中就标识卡页为脏,卡表也会记录卡页的开始地址，卡表也是用写屏障实现的；parNew在新生代维护一个大的卡表，G1在每个region取都维护一个卡表；


#### 安全点 ####
在做GC停顿的时候，需要代码执行到安全点才可以。安全点一般放在方法返回前，调用某个方法之后，抛出异常的位置，循环的结尾4个位置。
如何解决暂停的时候都让线程跑到安全点能，一般有两种方式，抢占式和主动中断。所以jvm在做GC的时候,为了保证代码的正确性.在代码的指定位置设置了安全点,在需要做GC中断用户线程的时候,不直接操作线程,而是设置了中断标记位,用户线程会不断主动轮询这个标志,如果标志为真,就在附近的安全点挂起,需要全部代码到达安全点才能进行GC（主动式中断）

抢占式中断：GC发生时，中断全部线程，如果线程不在安全点，恢复线程，继续执行直到附近安全点。

安全区域
线程sleep或者中断状态,安全区域指定一段代码中,引用关系不会发生变化

#### 垃圾回收器 ####
jvm自带不同的垃圾回收器，根据不同的情况，选择不同的回收器。因为每个回收器的侧重点和收集的算法不一样。

#### 1新生代的回收器 ####
新生代回收主要是复制算法；主要有Serial串行，ParNew，Parallel Scavenge；

Serial串行：-XX:+UseSerialGC -XX:+UseSerialOldGC；单线程,GC整个过程STW；新生代复制算法,老年代标记-整理

ParNew：可以理解成serial的多线程版本；-XX:UseParNewGC，-XX:ParallelGCThreads。多线程，整个过程STW。

Parallel Scavenge：线程数默认和CPU合数一样
;关注吞吐量,整个过程STW,jdk8默认该收集器；吞吐量=运行用户代码时间 /（运行用户代码时间 + 垃圾收集时间），主要适合在后台运算而不需要太多交互的任务，不能和CMS配合使用。

    -XX:+UseParallelGC 
    -XX:+UseParallelOldGC
    -XX:MaxGCPauseMillis
    -XX:GCTimeRatio
    -XX:+UseAdaptiveSizePolicy
    -XX:ParallelGCThreads 默认CPU核心数

#### 2老年代回收器 ####
CMS（cuncurrent mark sweep）-XX:+UseConcMarkSweepGC，并发标记清理,缩短停顿,提高用户体验,JDK8年龄默认是6进入老年代.标记清理算法.

CMS的执行流程分为初始化标记，并发标记，重新标记，并发清理和并发重置5个过程。
阶段1：初始化标记是STW，GC Root标记直接引用对象这一层。加速枚举GC Roots，使用oopMap的记录哪些是内存是引用，上面也提到了，同时也提到了GC roots + remember set.作为扫描的开始。

阶段2：并发标记，耗时比较长，而且和应用程序并行，注意CPU的负载，标记算法是上面提到的三色标记发

阶段3：重新标记，STW，上面提到了使用写屏障+增量更新或者原始快照的方式解决错标漏标的问题

阶段4：并发清理，把白色的对象清楚，线程数是（核心数+3)/4。

阶段5：并发重置，重置本次GC标记的数据

CMS的缺点：
CPU资源敏感，浮动垃圾，并发失败，碎片化。注意并发实现回导致使用串行回收器。CMS默认情况下92%执行清理，新的垃圾大于8%进不了老年代，停止并发，会使用串行收集器。如果出现并发失败，需要考虑调整CMS的触发比例值。

CMS回收相对新生代回收慢很多，原因在于：老年代空间大，存活多，垃圾不是连续的，并发可能失败，默认GC之后需要整理，整个过程耗时比较长,但停顿时间短


一些核心参

    
    -XX:+ConcGCThreads 并发GC线程,默认(核心+3) /4
	-XX:+CMSScavengeBeforeRemark 重新标记之前先执行youngGC.新生代可能引用了老年代,先执行新生代,降低CMS并发标记阶段的开销 
    -XX:+UseCMSCompactAtFullCollection 清理完成做整理
    -XX:CMSFullGCsBeforeCompaction=0 默认0每次都执行,多少次之后做整理
    -XX:CMSInitiatingOccupancyFraction=92 占用比例执行GC,配套参数使用
    -XX:+UseCMSInitiatingOccupancyOnly 使用用设定回收阈值,否则自动调整(配套上面参数,建议不指定)
 	-XX:+CMSParallelRemarkEnabled 重新标记时多线程执行(默认开启)
	-XX:+CMSParallelInitialMarkEnabled 初始化标记多线程(jdk8默认开启)

(X越多,表示越不稳定,以后可能废除)


#### 3G1收集器 ####
jdk7推出,jdk9官方推荐;关注点:暂停时间;针对多核大内存的机器;最大支持64G，逻辑划分eden,survivor,old,humongous4个区;复制算法。参数-XX:+UseG1GC

运行原理：
改变了管理内存的方法，把大块内存切分小块管理.老年代和新生代大小按需分配。基于复制算法。尽量确保回收停顿时间，跟踪region的回收价值，回收最快或者最多的region。平均分2048个region，一个region对应一个rememberSet,记录对象在每个分区的引用关系,避免扫描整堆,加速标记.也可以手动指定大小。从逻辑上也分为新生代和老年代和大对象（超过region的50%）.新生代也分为eden和survivor 8:1:1。默认新生代5%，不超过60%。大小是动态调整的。新生代GC，把幸存者存入S1的Region中，但对比parnew的区别是，根据GC的停顿时间，回收对应的Region。G1会跟踪对每个Region的回收需要时间，对一部分的region进行回收。新生代初始化5%，如果到了5%但是回收过段比MaxGCPauseMills小很多，会继续扩充，不触发GC；触发的条件是，新生代GC占用60%，mixed GC 老年代45%。
-XX:InitiatingHeapOccupancyPercent默认值是45%
回收的同时回收大对象。G1 yongGC是STW的，mixedGC需要经历4个过程。初始化标记（STW），并发标记，最终标记（STW），筛选回收（STW）。过程和CMS差不多。但是筛选回收是STW的，默认分为8次回收，按照回收收益值,优先回收.
回收-执行-回收-执行.回收的时候是把剩余的对象复制到新的区域
控制停顿.参数MaxGCPauseMillis。
如果没有多余的region区域进行复制，就会触发串行GC。

关于G1的Remembered Sets，可以看上面关于卡表部分。每个region都有一个Remembered Sets 。解决region和region之间的引用问题。官方给出的数据是占用堆小于5%，使用G1消耗更大内存,大部分是记账(accounting)结构， 大于6GB ,低于0.5 秒暂停时间。

参数
    -XX:+UseG1GC
    -XX:MaxGCPauseMillis 最大停顿时间
    -XX:G1HeapRegionSize
    -XX:G1NewSizePercent 初始新生代5%
    -XX:G1MaxNewSizePercent 新生代最大比例
    -XX:MaxGCPauseMills 默认200ms
    -XX:G1MixedGCCountTarget 回收次数，默认8
    -XX:G1HeapWastePercent 空闲region数停止默认5%
    -XX:G1MixedGCLiveThresholdPercent 存活低于该值85%才回收


8G以上,16G内存适用于G1，类似kafka,ES，统计,BI。优化的思路是合适设置XX:MaxGCPauseMills 太快可能导致GC频繁，太慢卡顿，survivor存不下幸存者进入老年代。


#### ZGC ####
jdk11推出,关注低延迟;停顿时间不超过10ms;最大16T;吞吐降低不超过15%. -XX:+UseZGC;在G1基础上.暂时不实现分代。不同的是使用读屏障,颜色指针实现可并发的标记-整理算法.将region划分小,中,大的容量。之前的GC信息保存在对象头,但ZGC保存在指针中.64位指针,低44位用于寻址,高位用于业务。还有一个比较厉害厉害消除统一内存访问架构.所有cpu对同一块内存访问,导致资源争抢.每个cpu绑定一块内存,且这块内存离这个cpu最近;自动感知并利用numa架构

执行流程：
并发标记：GC的标志在指针上,标记阶段会更新颜色指针;只有在mark start和mark end存在短暂的暂停；
并发预备重分配：计算region回收的效益
并发重分配：把存活对象复制到新region,使用读屏障把新的地址写到转发表中.下次读的时候,通过转发表获取新的地址
并发重映射：把原来旧对象的引用地址更新成新的地址

#### 日志demo ####

    0.137: [GC (Allocation Failure) 0.165: [ParNew: 3579K->512K(4608K), 0.0362486 secs] 3579K->1653K(9728K), 0.0644162 secs] [Times: user=0.03 sys=0.00, real=0.06 secs]
    
    0.137 发生时间: [GC (Allocation Failure GC原因) 0.165： 
	[ParNew: 3579K GC前->512K GC后(4608K 总大小), 0.0362486 secs 花费时间] 
    3579K GC前->1653K GC后(9728K整堆), 0.0644162 secs] [Times: user=0.03 sys=0.00, real=0.06 secs] 

[ParNew:  GC前->GC后( 总大小),  花费时间] GC前-> GC后(整堆)

    [CMS:8194K->6836K(10240K), 0.0049920 secs] 11356K->6836K(19456K), [Metaspace: 2776K->2776K(1056768K)], 0.0106074 secs]
    [CMS:GC前K->GC后K(10240K), 0.0049920 secs] 11356K->6836K(19456K), [Metaspace: 2776K->2776K(1056768K)], 0.0106074 secs]

#### 工具 ####
工具1：jstat

jstat查看运行情况，包括新生代老年代使用了多少内存，GC的次数等。使用该工具，可以查看每秒新增多少内存，对发生GC时间的一个预估，然后配置多少内存。可以按照内存篇的大小*频率*时间计算出多少时间后发生GC。

    jstat -gc PID
C表示容量,U已使用,T时间

- S0C：这是From Survivor区的大小
- S1C：这是To Survivor区的大小
- S0U：这是From Survivor区当前使用的内存大小
- S1U：这是To Survivor区当前使用的内存大小
- EC：这是Eden区的大小
- EU：这是Eden区当前使用的内存大小
- OC：这是老年代的大小
- OU：这是老年代当前使用的内存大小
- MC：这是方法区（永久代、元数据区）的大小
- MU：这是方法区（永久代、元数据区）的当前使用的内存大小
- YGC：这是系统运行迄今为止的Young GC次数
- YGCT：这是Young GC的耗时
- FGC：这是系统运行迄今为止的Full GC次数
- FGCT：这是Full GC的耗时
- GCT：这是所有GC的总耗时

    jstat -gc PID 1000 10

每秒执行一次，10次 ,看到新生代每秒增长多少，推算多久一次minorGC，minorGC总耗时/次数=评价耗时；如果想查看回收情况：
根据上面算得的增加，计算多久（9秒）一次GC
调整收集时间jstat -gc PID 9000 10。
更直接看GC日志


工具2：jstack查看线程，可以排查cpu飙升和死锁。
使用top -Hp pid查看当前的线程，按大写的P进行排序，然后把线程的pid转换成16进制，然后使用jstack 进程 | grep 0x60a -C5，查看里面的信息，包含了类的方法名。


工具3：jmap查看对象分布，官方建议用jcmd命令代替jmap

jmap -heap PID （新生代，老年代，S0，S1）多大，用了多少，剩余多少
jmap -histo PID 对象占用内存空间的大小降序排列
jmap -dump:live,format=b,file=dump.hprof PID 导出快照。
使用eclipse的MAT分析内存分布，可以排查内存飙升

如果是排查堆外内存，使用jcmd,查看内存篇的堆外内存；打开堆外内存跟踪，-XX:NativeMemoryTracking=detail，执行jcmp命令，jcmd pid VM.native_memory detail scale=MB > temp.txt 
或者使用google-perftools；


工具4：jinfo查看配置信息。更改配置之后查看是否生效
jinfo -flags pid 查看jvm参数
jinfo -sysprops pid 系统参数


打印GC
-XX:+PrintGCDetils：打印详细的gc日志 
-XX:+PrintGCTimeStamps：打印GC发生时间 
-Xloggc:gc.log：gc日志写入磁盘文件
-XX:GCLogFileSize=100M
-XX:+HeapDumpOnOutOfMemoryError  
-XX:HeapDumpPath=/usr/local/app/oom

监控
监控open-faIcon（小米）
zabbix（zabbix-java-gateway）
调优工具Arthas(阿里)

自研监控（MXBean）
java.lang.management.ManagementFactory


#### 优化 ####
优化的前提必须明确目标；性能指标（CPU,IO，内存，吞吐量，响应时间等），survivor能保存幸存者。尽量减少FullGC。

从性能维度关注延迟/抖动，吞吐量/TPS和系统容量，
性能的指标分为业务指标和资源（硬件）约束指标。
业务指标是指吞吐量,响应时间,并发数,业务成功数。
资源（硬件）约束指标是CPU，内存和IO

1CPU
CPU可以通过top命令或者其他工具查看，比如vmstat ；
vmstat 1

- us：用户占用CPU的百分比
- sy：系统(内核和中断)占用CPU的百分比
- id：CPU空闲的百分比
- r： 可运行进程数，包括正在运行(Running)和已就绪等待运行(Waiting)的进程
- b:睡眠的线程
- cs：每秒上下文切换次数

top命令
Load Average 1,5,15分钟的平均负载

lscpu
查看cpu个数
CPU的指标使用率通常用us + sy来计算，其可接受上限通常在70%~80%。

sy的值长期大于25%，应该关注in(系统中断)和cs(上下文切换)的数值

r的值等于系统CPU总核数，则说明CPU已经满负荷。在负载测试中，其可接受上限通常不超过CPU核数的2倍

2内存
vmstat 1
free：可用内存
si：每秒从SWAP读取到内存的数据大小
so：每秒从内存写入到SWAP的数据大小
指标：free可接受的范围通常应该大于物理内存的20%

3IO
iostat -dxk 1
r/s和w/s:每秒读写
rkB/s和wkB/s:每秒读/写的数据大小

iptraf -d eth0

指标
机械硬盘IOPS随机存储一般在100左右；SSD型号不一样差别很大，几千到几万之间
（Input/Output Operations Per Second）每秒读写次数


生产的标准
4核8G；压力测试QPS不能低于3万，数据库负载不能超过50%，服务器负载不能超过70%, 单次请求时长不能超过70ms，错误率不能超过5%；

一个4核8G内存的服务器，最大线程数800较为合适，最小工作线程数为100较为合适

#### 调优案例： ####
日请求上亿：
