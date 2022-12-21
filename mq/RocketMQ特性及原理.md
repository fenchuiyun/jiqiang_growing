### 消费发送

生产者向消息队列里写入消息，不同的业务场景需要生产者采用不同的写入策略。比如同步发送、异步发送、OneWay发送、延迟发送、发送事务消息等。默认使用的是DefaultMQProducer类，发送消息要经过五个步骤：

1. 设置producer的GroupName
2. 设置instanceName，当一个JVM需要启动多个Producer的时候，通过设置不同的instanceName来区分，不设置的话系统使用默认的名称“DEFAULT”。
3. 设置发送失败重试次数，当网络出现异常的时候，这个次数影响消息的重复投递次数。想保证不丢失消息，可以设置多重试几次。
4. 设置NameServer地址
5. 组装消息并发送。



消息发生返回状态（sendResult#SendStatus）有如下四种：

1. FLUSH_DISK_TIMEOUT
2. FLUSH_SLAVE_TIMEOUT
3. SLAVE_NOT_AVAILABLE
4. SEND_OK

不同状态在不同的刷盘策略和同步策略的配置下含义是不同的：

1. **FLUSH_DISK_TIMEOUT**：表示没有在规定时间内完成刷盘（需要Broker的刷盘策略被设置成SYNC_FLUSH才会报这个错误）。
2. **FLUSH_SLAVE_TIMEOUT**：表示在主备方式下，并且Broker被设置成SYNC_MASTER方式，没有在设定时间内完成主从同步。
3. **SLAVE_NOT_AVAIABLE**：这个状态产生的场景和FLUSH_SLAVE_TIMEOUT类似，表示在主备方式下，并且Broker被设置成SYNC_MASTER，但是没有找到被配置成Slave的Broker。
4. **SEND_OK**：表示发送成功，发送成功的具体含义，比如消息是否已经被存储到磁盘？消息是否被同步到了Slave上？消息在Slave上是否被写入磁盘？需要结合所配置的刷盘策略、主从策略来定。这个状态还可以简单理解为，没有发生上面列出的三个问题状态就是SEND_OK。

写一个高质量的生产者程序，重点在于对发送结果的处理，要充分考虑各种异常，写清对应处理的逻辑。

**提升写入的性能**

发送一条消息出去要经过三步

1. 客户端发送请求到服务器。
2. 服务器处理该请求。
3. 服务器向客户端返回应答。

一次消息的耗时是上述三个步骤的总和。

在一些对速度要求高，但是可靠性要求不高的场景下，比如日志收集类应用，可以采用OneWay方式发送

OneWay方式指发送请求不等待应答，即**将数据写入客户端的Socket缓冲区就返回**，不等待对方返回结果。

用这种方式发送消息的耗时可以缩短到**微秒级**。

另一种提高发送速度的方式是增加Producer的并发量，**使用多个Producer同时发送**，我们不用担心多Producer同时写会会降低消息写磁盘的效率，RocketMQ引入了一个并发窗口，在窗口内消息可以并发地写入DIrectMem中，然后异步地将**连续一段无空洞的数据**刷入文件系统中。

**顺序写**commitLog可让RocketMQ无论在HDD还是在SDD磁盘情况下都能**保持较高的写入性能**。

### 消费消息

简单总结消费的几个要点：

1. 消息消费方式（Pull和Push）
2. 消息消费的模式（广播模式和集群模式）
3. 流量控制（可结合sentinel来实现）
4. 并发线程设置
5. 消息的过滤（Tag、Key）TagA||TagB||TagC*null

当Consumer的处理速度跟不上消息的产生速度，会造成越来越多的消息积压，这个时候**首先查看消费逻辑本身有没有优化的空间**，除此之外还有三种方法可以提高Consumer的处理能力。

1. 提高消费并行率

   在**同一个ConsumerGroup**下(Clustering方式)，可以通过**增加Consumer实例**的数量来提高并行度。

   通过加机器，或者在已有机器中启动多个Consumer进程都可以增加Consumer实例数。

   注意：总的Consumer数量不要超过Topic下**Read Queue**数量，超过的Consumer实例接受不到消息。

   此外，通过提高单个Consumer实例中的**并行处理的线程数**，可以在同一个Consumer内增加并行度来提高吞吐量（设置方法是修改consumeThreadMin和consumerThreadMax）。

2. 以批量方式进行消费

   某些业务场景下，多条消息同时处理的时间会大大小于逐个处理的时间总和，比如消费消息中设计update某个数据库，一次update10条的时间会大大小于十次update1条数据的时间。

   可以通过批量方式消费来提高消费的吞吐量。实现方法是设置Consumer的consumeMessageBatchMaxSize这个参数，默认是1，如果设置为N，在消息多的时候，在消息多的时候每次收到的是个长度为N的**消息链表**。

3. 检测延时情况，跳过非重要消息

   consumer在消费的过程中，如果发现由于某种原因发生严重的消息堆积，短时间无法消除堆积，这个时候可以选择丢弃不重要的消息，是Consumer尽快追上Producer的进度。



### 消息存储

##### 存储介质

- 关系型数据库DB

  Apache下开源的另一款MQ-ActiveMQ（默认采用的KahaDB做消息存储）可选用JDBC的方式来做消息持久化，通过简单的xml配置信息即可实现JDBC消息存储。由于，普通关系型数据库（如MySql）在单表数据量达到千万级别的情况下，其IO读写性能往往会出现瓶颈。在可靠性方面，该种方案非常依赖DB，如果一旦DB出现故障，则MQ的消息就无法落盘存储导致线上故障。

- 文件系统

  目前业界较为常用的几款产品(RocketMQ/Kafka/RabbitMQ)均采用的消息刷盘至所部署虚拟机/物理机的文件系统来做持久化（刷盘一般可以分为异步刷盘和同步刷盘两种模式）。消息刷盘为消息存储提供了一种高效率、高可靠和高性能的数据持久化方式。除非部署MQ机器本身或是本地磁盘挂了，否则一般是不会出现无法持久化的故障问题。

##### 性能对比

文件系统>关系型数据库DB

##### 消息的存储和发送

1. 消息存储

   目前的高性能磁盘，顺序写速度可以达到600MB/s,超过了一般**网卡**的传输速度。

   但是磁盘的随机写的速度只有大概100KB/s，和顺序写的性能相差6000倍!

   因为有如此巨大的速度差别，好的消息队列系统会比普通的消息队列系统速度快多个数量级。

   RocketMQ的消息用**顺序写**，保证了消息存储的速度。

2. 存储结构

   RocketMQ消息的存储是由ConsumeQueue和CommitLog配合完成的，消息真正的物理存储文件是CommitLog，ConsumeQueue是消息的逻辑队列，**类似数据库的索引文件**，存储的是**指向物理存储的地址**。每个Topic下的每个Message Queue都有一个对应的ConsumeQueue文件。

<br>

<img src="https://fenchuiyun-scw.oss-cn-shanghai.aliyuncs.com/blog/image-20221121095424941.png" alt="image-20221121095424941" style="zoom:67%;" />

- 消息存储架构图中主要有下面三个跟消息存储相关的文件构成。

  1. CommitLog：消息主体以及元数据的存储主体，存储Producer端写入的消息主体内容，消息内容不是定长的。单个文件大小默认1G，文件名长度为20位，左边补零，剩余为起始偏移量，比如00000000000000000000代表了第一个文件，起始偏移量为0，文件大小为1G=1073741824，以此类推。消息主要是顺序写入日志文件，当文件满了，写入下一个文件；

     ![image-20221121101411316](https://fenchuiyun-scw.oss-cn-shanghai.aliyuncs.com/blog/image-20221121101411316.png)

  2. ConsumeQueue：消息消费队列，引入的目的主要是**提高消息消费的性能**

     RocketMQ是基于主题Topic的订阅模式，消息消费是针对主题进行的

     如果要遍历commitlog文件根据topic检索消息是非常低效的。

     Consumer即可根据ConsumeQueue来查找待消费的消息。

     其中，ConsumeQueue（逻辑消费队列）作为消费消息的索引：

     1. 保存了指定Topic下的队列消息在CommitLog中的起始物理偏移量offset
     2. 消息的大小size
     3. 消息tag的HashCode值

     consumequeue文件可以看成是基于topic的commitlog索引文件，故consumequeue文件夹的组织方式如下：

     topic/queue/file三层组织结构

     具体存储路径为：$HOME/store/consumequeue/{topic}/{queueId}/{fileName}

     ![](https://fenchuiyun-scw.oss-cn-shanghai.aliyuncs.com/blog/image-20221121103512185.png)

     consumequeue文件采取定长设计，**每个条目**共20个字节，分别为：

     1. 8字节的commitlog物理偏移量
     2. 4字节的消息长度
     3. 8字节tag hashCode

     单个文件由30W个条目组成，可以像数组一样随机访问每一个条目

     每个ConsumeQueue**文件大小约5.72M**;
  
  3. IndexFile：indexFile（索引文件）提供了一种**可以通过key或时间来查询消息**的方法。
  
     1. Index文件的存储位置是：\$HOME/store/index/\${fileName}
     2. 文件名fileName是以创建时的时间戳命名的
     3. 固定的单个IndexFile文件大小约为400M
     4. 一个IndexFile可以保存2000W个索引
     5. indexFile的底层存储设计为在文件系统中实现**HashMap结构**，故rocketMq的索引文件其**底层实现为hash索引**。
  
     ![image-20221121110102688](https://fenchuiyun-scw.oss-cn-shanghai.aliyuncs.com/blog/image-20221121110102688.png)
  
  
  
  

<br>

### 消息过滤

RocketMQ分布式消息队列的消息过滤方式有别于其他MQ中间件，是在**Consumer端订阅消息时再做消息过滤**的。

RocketMQ这么做是在于其Producer端写入消息和Consumer端订阅消息采用**分离存储的机制**来实现的，consumer端订阅消息时需要通过**ConsumeQueue**这个消息消费的逻辑队列拿到一个索引，然后再从**CommitLog**里面读取真正的消息实体内容，所以说到底也是绕不开其存储结构。

其ConsumeQueue的存储结构如下，可以看到其中有**8个字节**存储的**Message Tag的哈希值**，基于Tag的消息过滤正式基于这个字段值的。

![image-20221121111456132](https://fenchuiyun-scw.oss-cn-shanghai.aliyuncs.com/blog/image-20221121111456132.png)

主要支持如下2种的过滤方式

1. Tag过滤方式：Consumer端在**订阅消息**时除了指定Topic还可以指定TAG，如果一个消息有多个TAG，可以用｜｜分隔
   1. Consumer端会将这个订阅请求构建成一个SubscriptionData，发送一个Pull消息的请求给Broker端
   2. Broker端从RocketMQ文件存储层——Store读取数据之前，会用这些数据先构建一个MessageFilter，然后传给Store
   3. Store从ConsumeQueue读取到一条记录之后，会用它记录的消息tag hash值去做过滤
   4. 在服务端只是根据hashCode进行判断，无法精确对tag原始字符串进行过滤，在消息消费端拉取消息后，还需要对消息的原始tag字符串进行对比，如果不同，则丢弃该消息，不进行消息消费



### 零拷贝原理

##### pageCache

- 由内存中的物理page组成，其内容对应磁盘上的block。
- page cache的大小是动态变化的。
- backing store：cache缓存的存储设备
- 一个page通常包含多个block，而block不一定是连续的

**读Cache**

- 当内核发起一个读请求时，先会检查请求的数据是否缓存到了page cache中
  - 如果有，那么直接从内存中读取，不需要访问磁盘，此即cache hit（缓存命中）
  - 如果没有，就必须从磁盘中读取数据，然后内核将读取的数据再缓存到cache中，如此后续的请求就可以命中缓存了
- page可以只缓存一个文件的部分内容，而不需要把整个文件都缓存起来

**写Cache**

- 当内核发起一个写请求时，也是直接往cache中写入，后备存储中的内容不会直接更新
- 内核会将被写入page标记为dirty，并将其加入到dirty list中
- 内核会周期性地将dirty list中的page写回到磁盘上，从而使磁盘上的数据和内存中缓存的数据一致

**cache回收**

- Page cache的另一个重要工作是释放page，从而释放内存空间
- cache回收的任务是选择合适的page释放
  - 如果page是dirty的，需要将page写回到磁盘中再释放



##### cache和buffer的区别

1. Cache：缓存区，是高速缓存，是位于CPU和主内存之间的容量最小但速度最快的存储器，因为CPU的速度远远高于主内存的速度，CPU从内存中数量	需要等待很长的时间，而cache保存着CPU刚用过的数据或者循环使用的部分数据，这时从Cache中读取数据更快，减少了CPU等待的时间，提高了系统的性能。

   Cache并不是缓存文件的，而是缓存块的(块是I/O读写最小的单元)；Cache一般会用在I/O请求上，如果多个进程要访问某个文件，可以把此文件读入Cache中，这样下一个进程获取CPU控制权并访问此文件直接从Cache读取，提高系统性能。

2. Buffer：缓冲区，用于存储速度不同的设备或优先级不同的设备之间传输数据；通过buffer可以减少进程间通信需要等待的时间，当存储速度较快的设备与速度较慢的设备进行通信时，存储慢的设备先把数据存放到buffer，达到一定程度存储快的设备再读取数据，在此期间存储快的设备CPU可以干其他事情。

Buffer：一般是用在写入设备的，例如：某个进程要求多个字段被读入，当所有要求的字段被读入之前已经读入的数据将会被放到buffer中。

##### HeapByteBuffer和DirectByteBuffer

HeapByteBuffer,是在**jvm堆上的一个buffer**，底层的本质是一个数组，用类封装维护了很多的索引（limit/position/capacity等）。

DirectByteBuffer,底层的数据是维护在**操作系统内存**中，而不是jvm里，DirectByteBuffer里维护了一个引用address指向数据，进而操作数据。

HeapByteBuffer优点：内容维护在jvm里，把内容写进buffer里的速度更快；跟容易回收

DirectByteBuffer优点：跟外设（IO设备）打交道时会快很多，因为外设读取jvm堆里的数据时，不是直接读取的，而是把jvm里的数据读取到一个内存块里，再在这个快里读取，如果使用DirectByteBuffer，则可以省去这一步，实现zero copy（零拷贝）。

外设之所以要把jvm堆里的数据copy出来再操作，不是因为操作系统不能直接操作jvm内存，而是因为jvm在进行gc（垃圾回收）时，会对数据进行移动，一旦出现这种问题，外设就会出现数据错乱的情况。

<img src="https://fenchuiyun-scw.oss-cn-shanghai.aliyuncs.com/blog/image-20221205093209560.png" alt="image-20221205093209560" style="zoom:50%;" />



**所有的通过allocate方法创建的buffer都是HeapByteBuffer.**

<img src="https://fenchuiyun-scw.oss-cn-shanghai.aliyuncs.com/blog/image-20221205093354831.png" alt="image-20221205093354831" style="zoom:50%;" />

**堆外内存实现零拷贝**

1. 前者分配在JVM堆上（ByteBuffer.allocate()），后者分配在操作系统物流内存上（ByteBuffer.allocateDirect()，jvm使用C库中的malloc()方法分配堆外内存）；
1. DirectByteBuffer可以减少JVM GC压力，当然，堆中依然保存对象引用，fullgc发生时也会回收直接内存，也可以通过system.gc主动通知JVM回首，或者通过cleaner.clean主动清理。Cleaner.create()方法需要传入一个DirectByteBuffer对象和一个Deallocator（一个堆外内存回收线程）。GC发生时发现堆中的DirectByteBuffer对象没有强引用了，则调用Deallocator的run()方法回收直接内存，并释放堆中DirectByteBuffer的对象引用；
1. 底层I/O操作需要连续的内存(JVM堆内存容易发生GC和对象移动)，所以在执行write操作时需要将HeapByteBuffer数据拷贝到一个临时的（操作系统用户态）内存空间中，会多一次额外的拷贝。而DirectByteBuffer则可以省去这个拷贝动作，这是Java层面的“零拷贝”技术，在netty中广泛使用；
1. MappedByteBuffer底层使用了操作系统的mmap机制，FileChannel#map()方法就会返回MappedByteBuffer。DirectByteBuffer虽然实现了MappedByteBuffer，不过DirectByteBuffer默认并没有直接使用mmap机制。



##### 缓冲IO和直接IO

**缓冲IO**

缓存I/O又被称作为标准I/O，大多数文件系统的默认I/O操作都是缓存I/O。在Linux的缓存I/O机制中，数据先从磁盘复制到内核空间的缓冲区，然后从内核空间缓冲区复制到应用程序地址空间。

读操作：操作系统检查内核的缓冲区有没有需要的数据，如果已经缓存了，那么就直接从缓存中返回；否则从磁盘中读取，然后换存在操作系统的缓存中。

写操作：将数据从用户空间复制到内核空间的缓存中。这时对用户程序来说写操作就已经完成，至于什么时候再写到磁盘的由操作系统决定，除非现实地调用sync同步命令。

缓存I/O的优点：

1. 在一定程度上分离了内核空间和用户空间，保护系统本身的运行安全；
2. 可以减少读盘的次数，从而提高性能。

缓存I/O的缺点：

1. 在缓存I/O机制中，DMA方式可以将数据直接从磁盘读取到页缓存中，或者将数据从页缓存直接写回到磁盘上，而不能直接在应用程序地址空间和磁盘之间进行数据传输。数据在传输过程中就需要在**应用程序地址空间（用户空间）和缓存（内核空间）之间进行多次数据拷贝操作，**这些数据拷贝操作所带来的CPU以及内存开销是非常大的。

**直接IO**

直接IO就是应用程序直接访问磁盘数据，而不经过内核缓冲区，这样做的目的是减少一次从内核缓冲区到用户程序缓存的数据复制。比如说数据库管理系统这类应用，它们更倾向于选择它们自己的缓存机制，因为数据库管理系统往往比操作系统更了解数据库中存放的数据，数据库管理系统可以提供一种更加有效的缓存机制来提高数据库中数据的存取性能。

直接IO的缺点：如果**访问的数据不在应用缓存中，那么每次数据都会直接从磁盘加载**，这种直接加载会非常缓慢。直接IO和异步IO结合使用，会得到较好的性能。

下图分析了写场景下的DirectIO和BufferIO：

![image-20221220131730132](https://fenchuiyun-scw.oss-cn-shanghai.aliyuncs.com/blog/image-20221220131730132.png)

##### 内存映射文件 (Mmap)

在LIUNX中我们可以使用mmap用来在进程虚拟内存地址空间中分配地址空间，创建和物理内存的映射关系。

<img src="https://fenchuiyun-scw.oss-cn-shanghai.aliyuncs.com/blog/image-20221220132447999.png" alt="image-20221220132447999" style="zoom:67%;" />

**映射关系可以分为两种**

1. 文件映射 磁盘文件映射晋城的虚拟地址空间，使用文件内容初始化物理内存。
2. 匿名映射 初始化全为0的内存空间

**而对于映射关系是否共享又分为**

1. 私有映射（MAP_PRIVATE）多进程间数据共享，修改不反应到磁盘实际文件，是一个copy-on-write（写时复制）的映射方式。
2. 共享映射（MAP_SHARED）多进程间数据共享，修改反应到磁盘实际文件中。

**因此总结起来有4种组合**

1. 私有文件映射：多个进程使用相同的物理内存页进行初始化，但是各个进程对内存文件的修改不会共享，也不会反应到物理文件中
2. 私有匿名映射：mmap会创建一个新的映射，各个进程不共享，这种使用主要用于分配内存（malloc 分配大内存会调用mmap）。例如开辟新进程时，会为每个进程分配虚拟的地址空间。
3. 共享文件映射：多个进程通过虚拟内存技术共享同样的物理内存空间，对内存文件的修改会反映到实际物理文件中，他也是进程间通信（IPC）的一种机制。
4. 共享匿名映射：这种机制在进行fork的时候不会采用写时复制，父子进程完全共享同样的物理内存页，这也就实现了父子进程通行（IPC）。

**mmap只是在虚拟内存分配了地址空间，只有在第一次访问虚拟内存的时候才分配物理内存。**

在mmap之后，并没有在将内容加载到物理页上，只是在虚拟内存中分配了地址空间。当进程在访问这段地址时，通过查找页表，发现虚拟内存对应的页没有在物理内存中缓存，则产生“缺页”，由内核的缺页异常处理程序处理，将文件对应内容，以页为单位（4096）加载到物理内存，注意只是加载缺页，但也会受操作系统的一些调度策略影响，加载的比所需的多。

##### 直接内存读取并发送文件的过程

<img src="https://fenchuiyun-scw.oss-cn-shanghai.aliyuncs.com/blog/image-20221220140758408.png" alt="image-20221220140758408" style="zoom:67%;" />

##### mmap读取并发送文件的过程

<img src="https://fenchuiyun-scw.oss-cn-shanghai.aliyuncs.com/blog/image-20221220140824728.png" alt="image-20221220140824728" style="zoom67%;" />

##### sendFile零拷贝读取并发送文件的过程

<img src="https://fenchuiyun-scw.oss-cn-shanghai.aliyuncs.com/blog/image-20221220140947664.png" alt="image-20221220140947664" style="zoom:50%;" />

零拷贝（zero copy）小结

1. 虽然叫零拷贝，实际上sendfile有2次数据拷贝的。第一次时从磁盘拷贝到内核缓冲区，第二次是从内核缓冲区拷贝到网卡（协议引擎）。如果网卡支持SG-DMA(The Scatter-Gather Direct Memory Access) 技术，就无需从PageChache拷贝至Socket缓冲区；
2. 之所以叫零拷贝，是从内存角度来看的，数据在内存中没有发生拷贝，只是在内存和I/O设备之间传输。很多时候我们认为sendfile才是零拷贝，mmap严格来说不算；
3. Linux中的API为sendfile、mmap，Java中的API为FileChanel.transferTo()、FileChannel.map()等；
4. Netty、Kafka(sendfile)、RocketMq(mmap)、Nginx等高性能中间件中，都有大量利用操作系统零拷贝特性。

