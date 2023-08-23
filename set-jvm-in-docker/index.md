# Docker容器中设置jvm参数

<!--more-->

# 背景

* **java默认是通过/proc/meminfo来取服务器内存数据的**，而docker容器中/proc/meminfo文件记录的是宿主机的内存参数，jvm堆最大值默认是服务器的1/4，这样就可能会导致docker容器的限制内存最大值可能小于jvm堆内存上限。
* 现象就是在jvm触发fullGC或者majorGC之前堆内存就已经快达到容器最大内存限制，导致容器将java进程杀死。

# 解决方案

## 1、设置-Xmx最大内存

* 缺点就是每次更改容器最大限制内存都要更改一下对应-Xmx参数的值，显然这种是不合适的。

## 2、通过设置堆占内存比例的方式设置堆的最大内存值

* Docker是通过cgroup来实现最大内存设置的，添加识别cgroup配置的jvm参数即可读取到docker容器的内存最大值
* JDK 8u131+、JDK 9版本可以设置如下参数指定堆最大内存如下所示，其中MaxRamFraction默认值是4，堆内存的利用率比较低，可以适当调高该占比

  ```shell
  -XX:+UnlockExperimentalVMOptions #（开启实验性参数）
  -XX:+UseCGroupMemoryLimitForHeap # （指定JVM从主机读取cgroup限制）
  -XX:MaxRAMFraction=int #（服务器内存/jvm最大内存,默认是4,只能设置为整数，一般都是2）
  ```
* 不使用MaxRamFraction设置最大内存占比，而是将容器最大内存限制设置到环境变量中，在容器内通过环境变量读取到容器最大内存限制后通过-Xmx来设置堆最大内存为读取到的环境变量的80%，这样的好处是可以更灵活的配置堆的内存参数
* JDK 8u191+、JDK 10及以上版本可以使用如下参数来动态指定jvm堆内存最大值（**推荐**）
* ```shell
   -XX:+UseContainerSupport    #（使用容器内存）
   -XX:MaxRAMPercentage =75.0  #（使用容器内存百分比，表示jvm占容器内存75%）
  ```

