## Memory Allocation

### Driver Memory
- spark.driver.memory (JVM Memory)
- spark.driver.memoryOverhead (default value is 0.1, 10% of the spark.driver.memory) 

The minimum memoryOverhead is 384MB, so even when 10% of 1GB of driver memory is 100MB. Spark will still initiate 384MB of driver's memoryOverhead and 1GB of driver's memory, which in total is 1384 MB (~1.4G).

Your Spark driver uses all the JVM heap but nothing from the overhead.

The overhead memory is used by the container process or any other non JVM process within the container. 

![image](https://user-images.githubusercontent.com/59940078/170871183-434cd5d9-c131-4491-b779-c0ea9f55bf26.png)

### Executor Memory

The total memory allocated to the executor container is the sum of the following.

- **Overhead Memory**
- **Heap Memory**
- **Off Heap Memory**
- **Pyspark Memory**

Spark driver will ask for executor container memory using four configurations. 

- spark.executor.memoryOverhead: Overhead Memory (Default 10% of spark.executor.memory)
- spark.executor.memory: JVM Heap
- spark.memory.offHeap.size: Off Heap memory
- spark.executor.pyspark.memory: Pyspark memory

You will get an **OutOfMemory Exception** if you exceed either the Overhead Memory or the JVM Heap.

Before you ask for the driver or executor memory, you should check with your cluster admin (YARN, k8s ...) for the maximum allowed value.

![image](https://user-images.githubusercontent.com/59940078/170871837-d29ff579-84f3-4aed-9b26-fb6d9ba818df.png)


**If you request Executor's Memory that exceeds a node's Physical Memory Resources. The spark job will not run**

### Pyspark Memory
PySpark memory is irrelevant if your Spark application is written in Java or Scala. 

![image](https://user-images.githubusercontent.com/59940078/170871968-6e7ec12c-a04a-40f7-9d29-bb0a576f052c.png)

Since PySpark is not a JVM process, it will only get 800MB from the spark.executor.memoryOverhead (Image Above), of which ~300 to 400 MB is constantly consumed by the container processes and other internal processes. So you would get 400MB approximately because of the reduction from the container processes. If your PySpark consumes more than what can be accommodated in the overhead, you will see an OOM error.

Overhead Memory is used as your shuffle exchange or network read buffer in your Spark application.

## Memory Management

JVM Heap memoery is broken down into 3 parts:
- **Reserved Memory**
- **Spark Memory Pool**
- **User Memory**

![image](https://user-images.githubusercontent.com/59940078/182023951-992960e1-a98e-4aec-a624-7458118dfc51.png)

![image](https://user-images.githubusercontent.com/59940078/170874859-2ecd49e6-e7b9-4dbf-9334-d297d610a241.png)

### The Spark memory pool is further broken down 50:50 into two sub-pools 
- **Storage memory** (Caching Dataframe)
- **Executor memory** (Joining, sorting, grouping Operation for buffer - usually shuffles)

![image](https://user-images.githubusercontent.com/59940078/170875083-26bfd332-40b6-49c5-9dac-d2ee92b24cd7.png)

## CPU Allocation and Management

![image](https://user-images.githubusercontent.com/59940078/170875284-8c88135e-325b-4f6c-a624-65fd7241c658.png)

In this example, Executor will have four slots, and we can run four parallel tasks in these slots.
All these slots are threads within the same JVM. We have a single JVM, and all the slots are simply threads in the same JVM.

We have 4 threads to share 2 memory pools, and we will get 2310/4 (577.5M) of executor memory for each task. (Before Spark 1.6)

![image](https://user-images.githubusercontent.com/59940078/170875655-2e410c26-41a2-4293-8869-1e90357c87f5.png)

The **executor core** is critical because this configuration defines the **maximum number of concurrent threads**. **If you have too many threads, you will have too many tasks competing for memory**. **If you have a single thread, you might not be able to use the memory efficiently.**

In general, **Spark recommends two or more cores per executor**, but you should not go beyond five cores. More than five cores cause excessive memory management overhead and contention.

### Unified Memory Management (After Spark 1.6)

Assuming we have 4 slots but 2 active tasks. The unified memory manager can allocate all the available execution memory amongst the two active tasks. There is nothing reserved for any task.
The task will demand the execution memory, and the unified memory manager will allocate it from the pool.

If the **executor memory pool** is fully consumed, the memory manager can also allocate executor memory from the storage memory pool **as long as there is available space.**

![image](https://user-images.githubusercontent.com/59940078/170875721-986c0f50-96a4-4227-9582-3deda27af06d.png)

We start with a **50-50 boundary between executor memory and storage memory**. But you can change it using spark.memory.storageFraction. **Setting storage fraction will give a fixed amount of memory to the Storage Memory Pool (Hard Boundary).** (ex: 0.30 will use 30 for Storage and 70 left for the executor)

![image](https://user-images.githubusercontent.com/59940078/170875760-013c6cdd-6d17-40ce-9e85-d3959e5bb3e8.png)

**If you dont set Storage Fraction config**, this boundary is flexible at 50-50 boundary, and we can borrow space as long as we have free space.
So the memory manager can borrow some memory from the other side as long as the other side is free

### Example
Assuming we cached some dataframes and entirely consumed the storage memory pool.

![image](https://user-images.githubusercontent.com/59940078/170875898-f1435b91-351d-418d-817f-7ff1c710be55.png)

if we want to cache more dataframes, the memory manager will use extra memory from the executor memory pool since it was still available.

![image](https://user-images.githubusercontent.com/59940078/170875907-07e859c0-0370-45a1-82b4-13324e104934.png)

As we have cached dataframes, if the executor start performing **join operations (and operations that utilize the executor memory pool ...)**, therefore the executor needs memory to perform such operations, thus, the executor will start consuming memory as long as there is free space.

![image](https://user-images.githubusercontent.com/59940078/170875993-43b2ebff-6d83-4c0e-9620-ceb2e3706bf4.png)

**If there is still not enough space in the executor memory pool** to perform such operations, the **memory manager will start evict unnecessary cached data frames from the executor memory** that the storage memory pool has borrowed from before. 

![image](https://user-images.githubusercontent.com/59940078/170876148-e0352926-2a14-4221-aebc-19bd7836dbe6.png)

**However, if you still need more memory to perform such operations. The memory manager will try to evict some more cached dataframes from the storage memory pool and use the evicted available memory space to increase executor memory pool to perform operations.**

The unnecessary cached data frames will be evicted (spilled) to the the disk. However, if there are dataframes that cannot be spilled and the executor memory pool still need more memory from storage memory pool to run its operations, You would get an **OutOfMemory Exception**.

**If the Storage Fraction is already set before in the configurations then you cannot spill cached dataframes from the storage memory pool, even if there is available space. This is a hard boundary.**

## Spark Memory Configurations

![image](https://user-images.githubusercontent.com/59940078/170877119-2d2990d8-e4d1-47d6-9fd6-6afd922c8feb.png)

![image](https://user-images.githubusercontent.com/59940078/182024087-dd75ae29-c391-4dfa-b3d1-e1a07d595767.png)

Most of the spark operations and data caching are performed in the JVM heap. And they perform best when using the on-heap memory. However, the JVM heap is subject to garbage collection. 

If you are allocating a huge amount of heap memory to your executor, you might see excessive garbage collection delays.

**Using off-heap memory gives you the flexibility of managing memory by yourself and avoid GC delays.**
If you need an excessive amount of memory for your Spark application, it might help you take some off-heap memory. For large memory requirements, mixing some on-heap and off-heap might help you reduce GC delays.

By default, the off-heap memory feature is disabled.
You can enable it by setting spark.memory.offHeap.enabled = true.
You can set your off-heap memory requirement using spark.memory.offHeap.size.

Spark will use the off-heap memory to extend the size of spark executor memory and storage memory.

The off-heap memory is the extra space. So if needed, Spark will use it to buffer spark dataframe operations and cache the data frames. So adding off-heap is an indirect method of increasing the executor and storage memory pools.

![image](https://user-images.githubusercontent.com/59940078/170877357-df2a812f-9bb0-406a-a27d-92370ad8314c.png)

### PySpark Memory

If you are using PySpark, your application may need a Python worker. These Python workers cannot use JVM heap memory. So they use off-heap overhead memory. (ex: Functions (UDF) that are not native to Spark (from ther libraries)

![image](https://user-images.githubusercontent.com/59940078/170877406-73ddfb78-87fe-455b-8255-3a2431ae3352.png)

The spark.executor.pyspark.memory. is disabled by default because most of the PySpark applications do not use external Python libraries, and they do not need a Python worker. Setting this configurations will add to the Overhead Memory Pool on top of the default 10% Overhead memory pool from the Executor Memory.
