# spark-optimization
Spark Optimization techniques

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

 
