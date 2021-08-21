# MapReduce Reading Notes

**Created By: 蜜雪冰熊**

> 不动笔墨不读书

## 背景

Google内部每天会收集到海量的数据, 对这些数据的计算任务只能采用分布式, 从而能在一个合理的时间内结束这些任务, 但是如何并行化计算、分发数据、处理错误的问题导致了用大量复杂代码来掩盖原始的简单计算。
结果是Google设计了一种将并行化、容错、数据分布和负载均衡的细节隐藏的抽象，这些灵感来源于`Lisp`等函数式编程语言的`map`和`reduce`原语。

## 什么是MapReduce?

MapReduce是一个用于处理和生成大型数据集的一种编程模型和具体的相关实现; 具体表现在两个function, 一个是`map`，另一个是`reduce`
- 用户指定`map`函数来处理一个键值对从而生成一组中间键值对
- 用户指定`reduce`函数来将所有中间生成的键与对应的值合并

### 例子

> 题目: 统计一个集合里面的所有文档中每个单词出现的次数

```lisp
// key 文档名
// value 文档内容
map(String key, String value):
    for each word w in value:
        EmitIntermediate(w,"1")

// key 一个单词
// values 出现次数的一个列表
reduce(String key, Iterator values):
    int result = 0;
    for each v in values:
        result += ParseInt(v);
    Emit(AsString(result))
```

上面的只是以String为类型,下面写一个更通用的MapReduce形式
- map       (k1, v1)            ——> list(k2, v2)
- reduce    (k2, list(v2))      ——> list(v2)

可以看见输入和输出的可能来自不同的域, 但是中间生成的键值对和输出是同一个域

### 更多的例子

下面再放一些可以用`MapReduce`模型来计算的例子(这里直接用英语了)
具体可见本仓库目录下的`Lecture3.pdf`文件

- Distributed Grep


- Count of URL Access Frequency
  - `map`  (logs of web page requests)      ——> (URL,1)
  - `reduce`                                ——> (URL, total count)


- Reverse Web-Link Graph:
  - `map` (a link to target from a page named source)                  ——> (target, source)
  - `reduce` (target, list of all sources URLs)                        ——> (target, list(source))


- Term-Vector per Host: 总结了一个文档或者一组文档中出现的最重要的单词
  - `map`       ——> (hostname, term vector) hostname是从文档的URL中提取的
  - `reduce` (每个host的所有每个文档术语向量) ——> (hostname, term vector)

- Inverted Index: (倒排索引)
  - `map` (each document)     ——> (list(<word, document ID>))
  - `reduce` (all paris of given word)     将对应的document ID排序 ——> (word, list(document ID))

- Distributed Sort: (分布式排序)
  - `map`   extracts the key of each record       ——> (key, record)
  - `reduce`                                      ——> all pairs unchanged

## 实现MapReduce

不同环境下实现的MapReduce都是不尽相同的, Google这里展示的场景是: 与交换以太网连接的大型商业PC集群, 环境如下:
1. 都是双核的`X86`机器, 内存都在2-4GB
2. 带宽在100M bit/s - 1G bit/s之间
3. 一个集群有成百上千台机器,所以出错很正常
4. 存储器是由直接连接到单个机器上的廉价IDE磁盘提供的。内部开发的分布式文件系统用于管理存储在这些磁盘上的数据。文件系统使用复制在不可靠的硬件基础上提供可用性和可靠性
5. 用户将作业提交到一个调度系统, 每个作业由一组任务构成,由调度系统通过映射将其分发到一个集群内部的机器

### 执行流程

![MapReduce-figure1](https://github.com/code4EE/MIT6.824-Notes/blob/main/resources/images/mapreduce-figure1.jpg)

执行流程:
1. 程序中的MapReduce库首先将输入文件分割为`M`块, 每块的典型大小是`16MB - 64MB`(这个参数可以由用户控制), 然后启动集群机器上面的程序副本
2. `master`机器上面的程序副本有点特殊, 其余的机器都是被master分发工作的worker, 这些工作由`M`个map任务和`R`个reduce任务构成, master挑选空闲的worker来分发map任务或者reduce任务
3. 被分配map任务的worker读取对应文件的内容, 执行map之后中间产生的键值对都缓存在内存里面
4. 每隔一定时间这些缓存的键值对都会被写到本地磁盘, 被分隔函数分为`R`块, 这些被缓存的键值对的位置会被传递回master, master负责将这些位置告诉负责reduce的worker
5. 当负责reduce的worker被master通知以后, 使用`RPC`从负责map的worker的磁盘中读取缓存的数据, 读取完之后, reduce worker将这些数据按中间键排序 (排序这步是必须的, 因为不同的key会映射到同一个reduce worker, 此外如何中间生成的这些数据的量太大, 就要使用`外部排序`) 
6. reduce worker遍历所有的数据, 每遇到一个不同的key, 就将key和其对应的值传递给用户定义好的`Reduce`函数, `Reduce`函数的输出结果被添加到一个最终的文件(看上图)
7. 所有的`map` 和 `reduce` 任务都完成之后, master唤醒了用户程序, 这时, 用户程序中的`MapReduce`调用返回到了用户代码

完成成功之后, 执行结果就在`R`个输出文件里

### Master的数据结构

master有几种数据结构:
- 为每一个map和reduce任务存储状态(`idle` `in-progress` `completed`) 和 每一个worker机器的身份信息
- 作为一种map任务将中间文件的区域位置传递给reduce任务的渠道(管道)
### 容错

#### worker Failure

master会周期性地`ping`每一个worker, 如果超过规定的时间worker没有响应就会被master标记为是失败; 
任何map任务被完成后会被重置为`idle`状态, 任何处于`in progress`的map/reduce任务会被重启为`idle`状态
<br>
完成后的map任务会在`失败`的机器(就是和master ping不通的机器)上面重新执行因为他们的结果被存在失败的机器上从而不能被获取, 完成好的reduce任务不需要被重新执行因为它们的结果存放在全局的文件系统
<br>
如果一个map任务现在A上面执行再在B上面执行(因为主机ping A worker失败, 所以A被标记为失败), 那么所有执行reduce任务的worker都会被通知再执行, 任何没有从A读取到数据的reduce任务都会从worker B读取;

#### Master Failure

周期检查, 如果master任务死亡, 可以从上一个检查点的状态开始

### 本地化Locality

网络带宽是稀缺资源, 所以事实上输入数据是由GFS按照每个文件划分为64MB的块的形式并将这些块的副本存储在本地机器的磁盘上, 所以master拿到这些块的位置信息并将map任务分发给拥有该块副本的机器, 如果那个机器已经被标记为了失败, master尝试将map任务调度给离那个块副本近的机器(这个近可以理解为与失败的机器处于同一网络交换机下) , 所以当对一小部分worker进行大量的MapReduce操作, 大部分输入数据都在本地, 几乎不消耗网络带宽

### Task Granularity 任务的粒度

上面说过我们将map阶段分为M块, 将reduce阶段分为R块, 在具体实现中M和R右一定的取值界限, master必须做`O(M+R)`个决定并且保存`O(M*R)`个状态, 实际上内存占用是小的, 大概一对map/reduce任务只占一个`byte`的数据, 工程上, `M = 200, 000` | `R = 5,000` | `worker machine = 2,000`

### Back Task 备份任务

现象: 集群中经常会有一个游荡的机器, 可能该机器的磁盘损坏了, 由30MB/s下降到1MB/s, 再加上集群调度器在调度其他的机器导致它更加抢不到CPU、内存、本地磁盘和网络带宽, 最终导致整个MapReduce的操作消耗了很长的时间;<br>
解决: 将一个MapReduce的操作接近尾声的时候, master开始调度对正处于`in-progress`状态的任务进行备份执行, 当原来的task和备份的task中的任何一个先完成, 就算该任务完成; 这个备份任务的机制能够有效的减少整个大型MapReduce操作的完成时间.

## Refinements 改良

### Partitioning Function 分隔函数

默认的数据分隔函数是哈希(比如`hash(key) mod R`), 但是往往带来的是不平衡的分隔结果, 比如有时候输出的是URL, 我们想将同一个host的实体最终分在同一个文件中, 我们可以`hash(Hostname(urlkey) mod R)`

### Ordering Guarantees 顺序保证

将中间产生的键值对排序, 从而方便对输出文件的随机查找

### Combiner Function

比如前面统计所有文件中单词出现的次数, 很多文件都会产生`<hello, 1>`这种键值对, 然后直接将这一堆键值对通过网络发送给reduce任务, 但是我们可以在发送之前将这些键值对通过用户自定义的`Combiner Function`合并，该函数执行在每个进行map任务的机器上;
典型地, `combiner`函数和`reduce`函数的实现代码是一样的, 唯一的区别就是MapReduce操作如何处理函数的输出:
- reduce函数的输出写到输出文件
- combiner函数的输出写到将要发送到reduce任务的中间文件
部分组合有效的加快了MapReduce的操作

### Input and Output Types

MapReduce库提供了多种形式的输入数据的支持<br>
比如`text`输入模式下,将每一行看做一组键值对, <offset in the file, content of the line>, 当然用于也可以通过实现`reader`接口来增加对自定义输入类型的支持, 对待输出数据也是一样

### Side-effects 副作用

可以在map/reduce过程中生成辅助文件

### Skipping Bad Records

在MapReduce这个过程中我们可以不理会一些错误, 这些错误可能来自于第三方库，从而无法修改其代码, 每一个worker 进程都装载有一个捕获冲突和错误的handler, MapReduce在一个全局变量中存放了参数的序列号, 如果用户代码发送一个UDP报文信号给master, master在看见一特定的record有多个错误的时候就会指示下一次再执行响应的map/reduce任务的时候选择跳过这条record

### Local Execution

master都是动态将计算分配到成百上千的机器中, 所以当debug的时候就会变得困难, 所以后来实现了一个在本地机器上顺序执行所有作业的MapReduce库, 控制权交给用户

### Status Information

master运行了一个内部的HTTP服务器并且导出一组状态页供人类使用, 这些状态页记录了比如多少任务已经完成/正在执行, 输入/中间/输出数据的byte数等等, 此外顶级状态页记录了哪些worker失效了并且失效的时候正在执行哪些map或者reduce任务

### Counters

MapReduce库提供了一个计数功能, 比如可以统计总共处理了多少单词<br>
如何使用呢？在用户代码中创建一个命名的计数器对象, 然后用到的时候就调用`Increment()`增加
```lisp
Counter *uppercase;
uppercase = GetCounter("uppercase")

map(String name, String counters):
  for each word w in contents:
    if (IsCapitalized(w)):
      uppercase->Increment();
    EmitIntermediate(w, "1");
```
单个worker的计数器的值会周期性的发送给master, master收集好所有完成好的map/reduce任务的计数器的值并整合返回给用户代码, master也会消除因为备份的任务带来的重复计算得影响, 用户也能通过master状态页观察到整个计算的进度, 总而言之counter是有用的

## Performance

这部分介绍的是MapReduce在1TB数据里面搜索和排序的性能

## Conclusion 

MapReduce的应用很广泛








