[TOC]

# JVM介绍
## jvm生命周期
.java文件  编译成为  .class文件
.class文件生命周期
加载->验证->准备->解析->初始化->使用->卸载

* 加载： main方法入口
* 验证：符合jvm规范
* 准备：内存空间 null 0 
* 解析：符号引用替换为直接引用
* 初始化：设置的值

## 类加载器
Bootstrap ClassLoader, Extension ClassLoader, Application ClassLoader
* Bootstrap: jre里 lib目录核心类，例如：java.lang
* Extension： jre里 lib/ext 扩展类
* Application： 应用程序

双亲委派机制： 默认向上parent查找
可自定义classload打破双亲委派机制，例如tomcat
![5d9ae28d9239452f3b822dd81657e8e0.png](en-resource://database/1293:1)


# JVM内存分布
![e8994fe1d1ce49ef93a8c1ce3095bfc8.png](en-resource://database/1295:1)

## 元数据/方法区（Metaspace） 
永久代 ： 类信息；反射

## 线程栈（Thread）
### 程序计数器
### 虚拟机栈

## 堆（Heap）
### 年轻代：
复制整理算法
Eden区 两个Survivor区 默认8:1:1
### 老年代：
标记整理算法

### 年轻代 进入 老年代

1. 大对象直接分配老年代 默认为0 所有对象都在eden分配
2. 年轻代GC多次扔存活 默认15次
3. Minor GC后存活对象太多 无法放入Survivor区
4. Minor GC后触发动态年龄判定规则， 年龄1+年龄2+...+年龄n的对象总大小大于Survivor区大小的一半，则年龄n及以上的移动到老年代

### Minor GC触发时机
Eden区快满时

### Full GC触发时机
1. 老年代快满时 默认92%
2. Minor GC前， 老年代剩余大小 小于 预估值(Minor GC后进入老年代平均大小)
3. Minor GC后， Minor GC存活大于Survivor ，但是老年代剩余大小小于 Minor GC存活

# JVM常用参数
![e633b51649bfa9151e46980f65990947.png](en-resource://database/1297:1)

-Xms: jvm堆最小值
-Xmx: jvm堆最大值
-Xmn: jvm堆中新生代大小 剩余为老年代大小
-Xss: 每个线程的栈内存大小
-XX:MetaspaceSize:  元数据大小
-XX:MaxMetaspaceSize: 元数据最大大小

-XX:+PrintGCDetail 打印gc详细日志
-XX:+PrintGCTimeStamps  打印gc时间
-Xloggc:gc.log  gc日志写入文件

## 参数模板
4核8G服务器

-Xms4096M -Xmx4096M -Xmn3072M -Xss1M 
-XX:MetaspaceSize=256M -XX:MaxMetaspaceSize=256M
-XX:+UseParNewGC -XX:+UseConcMarkSweepGC
-XX:CMSInitiatingOccupancyFaction=92
-XX:UseCMSCompactAtFullCollection
-XX:CMSFullGCsBeforeCompaction=0

# JVM回收器
STW: Stop The World

## Serial / Serial Old
单线程
## Parallel Scavenge/ Parallel Old
并行 多核
[Parallel介绍](%3Ca href="https://www.cnblogs.com/rwxwsblog/p/6248205.html"%3Ehttps://www.cnblogs.com/rwxwsblog/p/6248205.html%3C/a%3E)
## ParNew / CMS
### ParNew(新生代)
-XX:+UseParNewGC
Eden区快满了触发 存活到S0 下次 Eden+S0 存活到S1

### CMS
* 初始标记： STW , GC Roots 去标记第一层
* 并发标记：多线程追踪
* 重新标记： STW，新产生，变化
* 并发清理：多线程

#### 浮动垃圾
并发清理期间产生 下次Full GC处理
#### Concurrent Mode Failure问题
默认老年代92%开始Full GC，CMS期间需要分配对象放不下（例如大对象直接分配），触发Concurrent Mode Failure，回退到Serial Old单线程STW慢慢回收，应避免这个问题
## G1
-XX:MaxGCPauseMills 默认200毫秒 每次GC的最大停顿时间
Region  每个Region预估时间 Region之间复制算法
Region在新生代 老年代之间可以转化
回收时间可控制：用最少的时间回收最多的对象
大堆的时候>4G 有优势

# JVM回收触发时机
GC Roots: 类在静态变量、方法的局部变量

* 强引用  例如 User user = new User() 
* 软引用 SoftReference  内存不够回收
* 弱引用 WeakReference
* 虚引用

# JVM经典案例
1. 问题：新增对象频繁且快速过期 minor GC后survive放不下 导致老年代频繁full GC
   方案： 扩充新生代大小
 2. 问题：select 没有where条件的大对象进入老年代 导致频繁Full GC
   方案：修改代码
 3. 问题：代码调用 System.gc()
   方案： -XX:+DisableExplicitGC 禁止显式GC
 4. 问题：CPU过高  线程多频繁切换/jvm频繁full gc
    方案：内存泄露导致对象无法回收等
 5. 问题：系统rpc调用慢 超时长导致内存泄露 Full GC频繁
    方案： 超时时间合理设置 增加处理速度
 6. 问题：调用MQ 失败放内存重试  qps高时堆积
    方案：快速失败放弃
 7. 问题： netty的direct buffer 
    方案：放开-XX:+DisableExplictitGC  netty需回收堆外
 8. 问题：log 切割string导致大量char[]对象 高并发时
    方案：减少log切割
 9. 问题： MQ一次获取过多对象到内存
    方案：阻塞队列接受MQ
# JVM优化
## 优化思路
尽量存活对象到Survivor区(小于50%) 进行minor GC 减少到老年代进行Full GC

## 计算方式
qps 每秒请求书 * 对象大小(字段总数* 字段大小) * 放到10-20倍（其他对象） = 每秒产生的对象大小
* 每秒占用多少内存？
* 多长时间触发一次Minor GC？
* 一般Minor GC后有多少存活对象？
* Survivor能放得下吗？
* 会不会频繁因为Survivor放不下导致对象进入老年代？
* 会不会因动态年龄判断规则进入老年代？

## 分析命令
jstat -gc PID 
jstat -gc PID 1000 10  分析新增和gc
jmap -heap PID 内存分布
jmap -histo PID | head 10 对象情况
jmap -dump:live,format=b,file=dump.hprof PID  内存快照

## 频繁Full GC原因
1. 高并发/处理数据量大->Young GC -> survivor过小 导致进入老年代
2. 过多大对象进入老年代
3. 内存泄露 大量对象无法回收 一直占用老年代
4. Metaspace 元数据区域加载类过多
5. 误调用 System.gc() 

## 内存泄露 与 内存溢出
### 内存泄露
代码问题导致引用无法消除内存区域无法回收 最终导致内存溢出
常见原因： 静态类、集合、大对象
### 内存溢出
1. metaspace 溢出
   没设置metaspace大小参数或者cglib动态生成过多类
2. 线程虚拟机栈溢出
   方法递归调用 (1M6000+次调用)
3. 堆内存溢出
   创建对象多过 gc来不及