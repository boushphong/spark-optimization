## Spark Cluster and Runtime Architecture

![image](https://user-images.githubusercontent.com/59940078/170861592-b321bc32-9d49-46ab-81f1-ff78f30b8372.png)


The driver itself does not perform any data processing work. The application driver will create executors inside worker nodes and utilize the computation resources from the worker nodes. (distribute the work to worker nodes) 

Driver will communicate with the Resource Manager (Yarn, Kubernetes, Mesos ...) and asks for containers to run computation on the worker nodes.
Driver will assign work to the executors, monitor them and manage the overall application.
Driver is also a **JVM application** just like the Executors

![image](https://user-images.githubusercontent.com/59940078/170861728-3b235a79-9b69-4926-a6ba-d70d93d9ef32.png)

Every JVM containers' resources is configured manually, and the resources in the JVM containers cannot be configured in a way that the containers resources are larger than the node resources.


### Spark architecture if not running pure SPARK resources such as custom Python UDF, Libraries ...
![image](https://user-images.githubusercontent.com/59940078/170861902-5b9180ba-2678-4aa0-8a20-cf6f8bcede20.png)


## Spark-submit 
![image](https://user-images.githubusercontent.com/59940078/170862229-cda5794a-6270-47fd-9087-fb218e0fb67a.png)

--conf: Memory Overhead
--driver-cores: number of cores for the driver
--driver-memory: amount of ram for the driver
--num-executors: number of executors that the **RM** will create
--executor-cores: number of cores for the each executor
--executor-memory: amount of ram for each executor memory

## Spark Deploy Mode
![image](https://user-images.githubusercontent.com/59940078/170862795-876cce7e-424f-4fb0-9dca-939653cd5b2a.png)

## Spark Jobs - Stage, Shuffle, Task, Slots
![image](https://user-images.githubusercontent.com/59940078/170862935-80d58db3-b445-4e40-b20e-ec41c51e7683.png)

Transformations , especially read operations can behave in two ways according to the arguments you provide

Lazily evaluated --> It will be performed only when an action is called
Eagerly evaluated --> A job will be triggered to do some initial evaluations

In case of read.csv()

If it is called without defining the schem a and inferSchema is disabled, it determines the columns as string types and it reads only the first line to determine the names (if header=True, otherwise it gives default column names) and the number of fields. Basically it performs a collect operation with limit 1 --> That's why you can see the first job.

**Each action triggers a Spark Job**

![image](https://user-images.githubusercontent.com/59940078/170863493-42b89923-7711-44ea-9467-b38c5aeebcbb.png)

![image](https://user-images.githubusercontent.com/59940078/170863809-a0de66cc-09a4-4675-afe2-897d01165c9b.png)

Since at Stage 1. The wide transformation (repartition method) partitions the data into 2 partitions. During the shuffle period in the network, the read exchange from Stage 2 will read in 2 partitions from Stage 1. Thus, We can run transformations on 2 partitions in 2 parralel tasks. The same processes would happen to Stage 3 as well.

Order of a Spark Job.
- **Job level:** Spark creates a job for each action. (A job may contains series of transformations). The Spark engine will optimize those transformations and create a logical plan for the job.
- **Stage level:** Then spark will break the logical plan at the end of every wide dependency and create two or more stages. If you do not have any wide dependency, your plan will be a single-stage plan. But if you have N wide-dependencies, your plan should have N+1 stages.
- **Shuffle/Sort level:** Data from one stage to another stage is shared using the shuffle/sort operation.
- **Task level:** Now each stage may be executed as one or more parallel tasks. The number of tasks in the stage is equal to the number of input partitions.

Example from images above: 
- In our example, the first stage starts with one single partition.
- So it can have a maximum of one parallel task.
- We made sure two partitions in the second stage.
- So we can have two parallel tasks in stage two.

Tasks:
- A task is the smallest unit of work in a Spark job.
- The Spark driver assigns these tasks to the executors and asks them to do the work.
- The executor needs the following things to perform the task. (Task Code and Data Partition)
- So the driver is responsible for assigning a task to the executor.
- The executor will ask for the code or API to be executed for the task.
- It will also ask for the data frame partition on which to execute the given code.
- The application driver facilitates both these things to the executor, and the executor performs the task.

## Slot
![image](https://user-images.githubusercontent.com/59940078/170865385-c95da582-21b7-4079-bd4d-9e1e1b098010.png)

- Each JVM (only 1 JVM per node) is given the resources that is asked by the driver. In this case, 4 CPU cores on each JVM container. So on every JVM on the node, there can be 4 executor slots which correspond to 4 parallel threads (4 Tasks running at the same time on a node).
- The driver knows how many slots do we have at each executor. And it is going to assign tasks to fit in the executor slots.

**After all the transformation, action will requires each task to send data back to the driver over the network and present back to the user**

## Task failure
- The driver considers the job done when all the tasks are successful.
- If any task fails, the driver might want to **retry** it.
- So it can restart the task at a **different executor**.
- **If all retries also fail**, then the driver returns an **exception** and marks the job failed.
