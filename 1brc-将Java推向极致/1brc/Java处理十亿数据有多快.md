# 将Java推向极限：在2秒内处理10亿行数据！

## Java并行流 Parallel Stream

        并行流是Java Stream API的一个特殊形式，它允许将一个数据流分成多个子流，并在不同的线程上同时处理这些子流。最终，这些子流的处理结果会被合并成一个最终结果。这种并行处理的方式可以充分利用现代多核处理器的计算能力，从而显著提高数据处理的速度。

### 并行流的使用

```java
// 伪代码展示并行流执行流程
public class ParallelStreamWorkflow {
    public static void main(String[] args) {
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8);
        
        // 并行流执行过程
        numbers.parallelStream()
            .map(n -> {
                // 任务拆分：ForkJoinPool 将任务分解为子任务
                System.out.println(Thread.currentThread().getName() + " processing: " + n);
                return n * 2;
            })
            .collect(Collectors.toList()); // 结果合并
    }
}
```

### 工作原理

#### 1. Fork/Join框架

并行流的底层实现依赖于Java 7引入的Fork/Join框架。Fork/Join框架通过递归的方式将任务不断**拆分成更小的任务**，直到这些任务小到足以被单独执行。然后，这些任务会被分配到不同的线程上并行执行。最后，这些任务的结果会被合并成一个最终结果。

#### 2. 工作窃取模式

Fork/Join框架采用了一种称为“工作窃取”的线程模式。在这种模式下，每个线程都会维护一个自己的任务队列。当线程完成自己的任务后，它会尝试从**其他线程的任务队列中“窃取”任务来执行**。这种机制有助于保持线程的忙碌状态，从而提高整体的执行效率。



> 怎么拆分？
> 怎么窃取？



#### 任务拆解的核心组件：Spliterator

    并行流的任务拆解依赖于一个关键接口：`Spliterator`（可分割迭代器）。这个接口是Java 8引入的，专门为并行处理设计。

```java
public interface Spliterator<T> {
    // 尝试分割当前元素集，返回新的Spliterator
    Spliterator<T> trySplit();
    
    // 处理剩余元素
    boolean tryAdvance(Consumer<? super T> action);
    
    // 估算剩余元素数量
    long estimateSize();
    
    // 特性标志（有序、大小已知等）
    int characteristics();
}
```



![](D:\Users\Desktop\技术分享\1brc-将Java推向极致\1brc\任务拆解.png)

**实际拆解过程（以ArrayList为例）**

```java
static final class ArrayListSpliterator<E> implements Spliterator<E> {
    private final ArrayList<E> list;
    private int index; // 当前索引
    private int fence; // 结束索引（-1表示未初始化）
    private int expectedModCount; // 用于并发修改检查

    public Spliterator<E> trySplit() {
        int lo = index, mid = (lo + fence) >>> 1; // 计算中点
        return (lo >= mid) ? null : // 无法分割
            new ArrayListSpliterator<>(list, lo, index = mid, expectedModCount);
    }
}
```

#### 拆解过程示例

假设有10个元素的ArrayList：

1. 初始Spliterator：index=0, fence=10

2. 第一次trySplit：mid=(0+10)/2=5 → 新Spliterator(0-5)，原Spliterator变为(5-10)

3. 递归分割直到任务足够小（默认阈值=1024元素）

#### ForkJoin框架的任务调度

```java
// 简化的ForkJoinTask执行逻辑
public abstract class ForkJoinTask<V> {
    final void doExec() {
        if (status >= 0) {
            boolean completed;
            try {
                completed = exec(); // 执行实际计算
            } finally {
                status = completed ? NORMAL : EXCEPTIONAL;
            }
        }
    }
    
    protected abstract boolean exec();
}

// 并行流使用的具体实现
static final class ForEachTask<S, T> extends CountedCompleter<Void> {
    public void compute() {
        Spliterator<S> rightSplit = spliterator.trySplit();
        if (rightSplit != null) {
            // 创建子任务
            ForEachTask<S, T> leftTask = new ForEachTask<>(this, rightSplit);
            ForEachTask<S, T> rightTask = new ForEachTask<>(this, spliterator);
            
            // 添加待处理任务数
            addToPendingCount(1);
            
            // 异步执行子任务
            leftTask.fork();
            rightTask.compute(); // 当前线程处理右侧任务
        } else {
            // 直接顺序处理
            leafAction.perform(spliterator);
        }
    }
}
```

#### 工作窃取（Work-Stealing）机制

1. **线程私有队列**：每个工作线程有自己的双端队列 (ForkJoinPool.WorkQueue)

2. **任务获取顺序**：
   
   - 从自己队列头部取任务（LIFO）
   
   - 当自己队列空时，从其他线程队列**尾部**窃取任务（FIFO）

3. **优势**：
   
   - 减少线程竞争
   
   - 保持工作负载均衡
   
   - 充分利用CPU资源

![](D:\Users\Desktop\技术分享\1brc-将Java推向极致\1brc\任务窃取.png)

**不同集合的拆解策略**

| 集合类型       | 拆解效率  | 原因             |
| ---------- | ----- | -------------- |
| ArrayList  | ★★★★★ | 基于数组，随机访问，平均分割 |
| HashSet    | ★★★★☆ | 基于哈希表，桶分割      |
| TreeSet    | ★★★☆☆ | 基于红黑树，树结构分割    |
| LinkedList | ★★☆☆☆ | 顺序访问，分割需遍历     |
| 文件IO流      | ★☆☆☆☆ | 顺序访问，难以高效分割    |

### 使用场景

![](D:\Users\Desktop\技术分享\1brc-将Java推向极致\1brc\何时推荐使用并行流.png)



- CPU密集型操作

- 大数据集（>10万条）

- 可并行化的独立任务

**性能关键因素：**

- **NQ模型**：N（数据量） × Q（操作耗时）值越大，并行收益越高

- **数据分割成本**：ArrayList 比 LinkedList 更适合并行

- **合并操作成本**：reduce/collect 的合并函数复杂度影响性能

### 注意事项

**线程安全问题**

```java
// 错误示例：共享可变状态()
List<String> results = Collections.synchronizedList(new ArrayList<>());
data.parallelStream().forEach(s -> results.add(process(s)));

// 正确做法：使用线程安全收集器
List<String> safeResults = data.parallelStream()
    .map(this::process)
    .collect(Collectors.toList());



// 危险：共享可变状态
AtomicInteger counter = new AtomicInteger();
data.parallelStream().forEach(item -> {
    counter.incrementAndGet(); // 虽然原子但性能差
    process(item);
});

// 推荐：无状态操作+线程安全收集
int total = data.parallelStream()
    .mapToInt(item -> {
        process(item);
        return 1; 
    })
    .sum();
```

- 虽然`Collections.synchronizedList`确保了单个`add`操作的线程安全，但在**并行流**中大量线程频繁调用`results.add()`时，会引发激烈的**锁竞争**。

- 每次`add`操作都需要获取/释放锁，高并发场景下会严重拖慢性能，抵消并行流的优势。



- **自动线程安全合并**：`Collectors.toList()`在并行流中自动将结果拆分到多个中间容器（避免锁竞争），最后合并到最终列表。

- **无副作用**：符合函数式风格，代码更简洁、可读性更高。

- **更高效**：减少同步开销，充分利用并行性能。



**选择数据结构**

```java
// ArrayList vs LinkedList 并行性能对比
List<Integer> arrayList = new ArrayList<>(10_000_000); // 随机访问快
List<Integer> linkedList = new LinkedList<>();         // 顺序访问慢

// ArrayList并行效率高3-5倍
arrayList.parallelStream().sum(); 
```

**避免自动装箱**

```java
// 差：频繁装箱拆箱
List<Integer> list = IntStream.range(0, 1_000_000).boxed().collect(Collectors.toList());
long sum = list.parallelStream().mapToLong(Integer::longValue).sum();

// 优：原始类型流
long sum = IntStream.range(0, 1_000_000).parallel().sum();
```

**任务拆分粒度**

```java
// 使用批量处理减少任务数
AtomicInteger batchCounter = new AtomicInteger();
Map<Integer, List<Data>> batches = largeDataSet.parallelStream()
    .collect(Collectors.groupingBy(
        data -> batchCounter.getAndIncrement() / 1000 // 每1000条一批
    ));

batches.values().parallelStream().forEach(batchProcessor::process);
```

**避免嵌套并行**

```java
// 危险：嵌套并行导致线程耗尽
data.parallelStream().forEach(item -> {
    item.getSubItems().parallelStream() // 嵌套并行流
        .forEach(this::process);
});

// 推荐：展平数据结构
data.parallelStream()
    .flatMap(item -> item.getSubItems().stream())
    .forEach(this::process);
```

## Java中的内存映射

        Java 中原生读写方式大概可以被分为三种：普通 IO，FileChannel（文件通道），mmap（内存映射）。区分他们也很简单，例如 FileWriter,FileReader 存在于 java.io 包中，他们属于普通 IO；FileChannel 存在于 java.nio 包中，也是 Java 最常用的文件操作类；而今天的主角 mmap，则是由 FileChannel 调用 map 方法衍生出来的一种特殊读写文件的方式，被称之为内存映射。

🔁 传统文件 I/O 的工作机制
假设你要从磁盘读取一块文件数据，传统方式大致流程如下：

1. -程序执行 read() 方法。

2. 操作系统会将磁盘上的数据 拷贝到内核缓冲区（kernel buffer）。

3. 然后再从 内核缓冲区拷贝到你的 Java 程序堆内存中。

4. 写文件的时候，再反过来重复一遍这个过程。

⚠️ 问题在哪里？

1. 有两次内存拷贝；

2. read/write 需要用户态 ↔ 内核态切换；

**mmap 的使用方式：**

```java
FileChannel fileChannel = new RandomAccessFile(new File("db.data"), "rw").getChannel();
MappedByteBuffer mappedByteBuffer = fileChannel.map(FileChannel.MapMode.READ_WRITE, 0, filechannel.size();
```

**关键参数与模式：**

- `FileChannel.MapMode`:
  
  - `READ_ONLY`: 只读映射。尝试写入会抛`ReadOnlyBufferException`。
  
  - `READ_WRITE`: 读写映射。修改会反映到页缓存和最终的文件（异步或显式`force()`后）。
  
  - `PRIVATE` (Copy-on-Write): 写时复制。修改只影响当前进程的缓冲区副本，不会写回原文件。适用于需要临时修改但不希望改变原文件的场景。

- `position`: 映射开始的文件位置。

- `size`: 映射区域的大小。

**MappedByteBuffer** 便是 Java 中的 mmap 操作类。

```java
// 写
byte[] data = new byte[4];
int position = 8;
// 从当前 mmap 指针的位置写入 4b 的数据
mappedByteBuffer.put(data);
// 指定 position 写入 4b 的数据
MappedByteBuffer subBuffer = mappedByteBuffer.slice();
subBuffer.position(position);
subBuffer.put(data);

// 读
byte[] data = new byte[4];
int position = 8;
// 从当前 mmap 指针的位置读取 4b 的数据
mappedByteBuffer.get(data)；
// 指定 position 读取 4b 的数据
MappedByteBuffer subBuffer = mappedByteBuffer.slice();
subBuffer.position(position);
subBuffer.get(data);
```

#### 优势：

- ✅ 操作系统**将文件直接映射到虚拟内存空间**；
- ✅ 数据访问由操作系统负责缓存，**避免了频繁 I/O 切换**；
- ✅ 支持大文件的高性能读写，**减少内存拷贝次数**；
- ✅ 更适合**并发读写和大文件合并**。

### “调整”我们的代码

#### 降低循环开销，提高循环的并行度，更充分的利用流水线来提升性能

        现在不管是ARM还是x86处理器，基本上都是超标量处理器。硬件资源例如MAC和ALU会有多个。超标量流水线也很常见，通常一个周期可以发射多条指令。对循环的优化通常可以做循环展开，我们做循环展开的目的就是为了能够充分使用这些硬件资源，填满pipeline，提升硬件资源的利用率。

示例循环

```java
int acc = 0;
for(int i = 0;i< 1000; i++)
{
    acc += data[i];
}
```

初次展开(存在依赖)

```java

int acc = 0;
for(int i = 0; i < 1000 / 4; i += 4)
{
  acc += data[i];
  acc += data[i + 1];
  acc += data[i + 2];
  acc += data[i + 3];
}
```

去除依赖

```java

int accRes = 0;
int acc[4] = {0};
for(int i = 0; i < 1000 / 4; i+=4)
{
  acc[0] += data[i];
  acc[1] += data[i + 1];
  acc[2] += data[i + 2];
  acc[3] += data[i + 3];
}
accRes = acc[0] + acc[1] + acc[2] + acc[3];
```

        需要注意的是的如果循环本身比较简单，开启O3有的交叉编译器会主动做循环展开。特别是一些dsp平台的交叉编译器，有的时候让循环尽量简单，让编译器去做比我们自己去做要好。所以当你循环展开没有效果，可能是编译器已经帮你做了。

#### 去除内存引用

        避免不必要的内存访问，尽量让数据待在寄存器里，我们直接从寄存器读写数据，最后再写出。

```java
// 这里每次循环都要访问内存
for(int i = 0; i < 1000; i += 2)
{
  arrA[100] += data[i];
  arrB[50] += data[i + 1];
}


// 去除内存引用
int a = 0;
int b = 0;
for(int i = 0; i < 1000; i += 2)
{
  a += data[i];
  b += data[i + 1];
}
arrA[100] += a;
arrB[50] += b;
```

#### 避免分支语句

        现在的处理器基本都支持分支预测，**分支预测失败**会导致流水线清空重排，带来性能损失，在循环内部应尽量避免分支判断。

循环内部有分支

```java

for(int i = 0; i < 1000; i++)
{
  if(a > b)
  {
    code......
  }
  else
  {
    code.....
  }
}
```

去除分支判断

```java

if(a > b)
{
  for(int i = 0; i < 1000; i++)
  {
    code......
  }
}
else
{
  for(int i = 0; i < 1000; i++)
  {
    code......
  }
}
```

同样我们也可以采取分段循环的方式来避免内部循环边界判断。

```java
// 每次循环都要进行边界判断
for(int i = 0; i < 1000; i++)
{
  if(i < 3)
  {
    code....
  }
  else if(i > 500)
  {
    code....
  }
  else
  {
    code....
  }
}



// 分段循环
for(int i = 0; i < 3; i++)
{
   code.... 
}
for(int i = 3; i <500; i++)
{
   code.... 
}
for(int i = 500; i < 1000; i++)
{
   code.... 
}
```

#### 循环读写合并

读写合并可以去除一些内存访问的开销，对于功耗和性能提升都有利

```java
// 冗余的内存读写

int tmp[1000]
for(int i = 0; i < 1000;i++)
{
  code....
  tmp[i] = data0[i] * data1[i];
}
for(int i = 0; i < 1000; i++)
{
  out[i] = tmp[i] + data2[i];  
}


// 合并内存读写
for(int i = 0; i < 1000; i++)
{
  code....
  out[i] = data0[i] * data1[i] + data2[i];
  code....
}
```

#### 查表优化

有些计算结果可以提前算好存在表里，后续直接查表，不用每次都去计算。

```java
// 每次循环都要做一次指数运算
// 假设data这个buffer的值的范围为0-255；
for(int i = 0; i < 1000; i++)
{
  res[i] = exp(data[i]) * gain[i];
}


// 提前计算好结果存在表里
for(int i = 0; i < 255; i++)
{
  table[i] = exp(i);
}
for(int i = 0; i < 1000; i++)
{
  res[i] = table[data[i]] * gain[i];
}
```

![题目](D:\Users\Desktop\技术分享\1brc-将Java推向极致\1brc\题目.png)

![规则说明](D:\Users\Desktop\技术分享\1brc-将Java推向极致\1brc\规则说明.png)

> 结果是通过在Hetzner AX161专用服务器（32核AMD EPYC™7502P (Zen2), 128 GB RAM）上运行程序确定的。
> 
> 程序从RAM磁盘运行（也就是说，从磁盘加载文件的IO开销无关紧要），使用机器的8个内核。每个竞争者必须通过1BRC测试套件（/test.sh）。超细程序用于测量所有条目的启动脚本的执行时间，即测量端到端时间。每个竞争者都要连续跑五次。最慢和最快的跑步会被丢弃。其余三次运行的平均值是该竞争者的结果，并将被添加到上面的结果表中。完全相同的measurement .txt文件用于评估所有竞争者。有关评估步骤的确切实现，请参阅脚本evaluate.sh。



**常规业务优化处理方案**

1.单线程处理文件改用并行流处理

2.采用更适合的数据结构 使用int代替double, 在纯计算场景,尽可能使用位运算

3.RandomAccessFile 文件均等分割并行处理   根据处理器数量 多线程处理

**非常规优化**

4.java 的mmap   FileChannel.map()

5.unsafe代替字节缓冲区进行读取文件

原因：即使是字节缓存区，仍然对读取文件有校验操作

6.自定义哈希表

- 指定大小：不需要考虑容量扩张，因为比赛的气象站大小已经确定
- 开放地址法：如果hash值对应的槽位（slot）已经存在，那么一直向右移动到第一个空的槽位
- 计算hash值和判断key是否相等：直接通过读取文件对应字节计算和判断，避免创建String对象



![image-20250618212951255](D:\Users\Desktop\技术分享\1brc-将Java推向极致\1brc\版本优化条目.png)




