## 简介

对golang程序进行性能分析，我们通常使用官方提供的pprof工具。pprof通常可以收集cpu profile，heap profile，trace，以及goroutine状态等信息。使用pprof通常需要以下几个步骤：

1. 引入http pprof web服务(import _ "net/http/pprof")
2. 获取profile文件(curl localhost:$PORT/debug/pprof/$PROFILE_TYPE)
3. 使用`go tool pprof`  进行分析

以下为一些pprof相关的资源：

* 配置pprof http服务：https://golang.org/pkg/net/http/pprof/
* 通过代码生成profile文件：https://golang.org/pkg/runtime/pprof/
* https://github.com/google/pprof （官方文档，有介绍pprof读取perf.data文件）
* 自定义proof profile类型:https://rakyll.org/custom-profiles/



## profile类型

profile文件记录了程序内部资源分配等信息，使用浏览器访问localhost:$PORT/debug/pprof/$可以查看所有可用的profile列表,以下为go pprof预定义的profile类型：

Profile Descriptions:

- **allocs**: A sampling of all past memory allocations
- **block**: Stack traces that led to blocking on synchronization primitives
- **cmdline**: The command line invocation of the current program
- **goroutine**: Stack traces of all current goroutines
- **heap**: A sampling of memory allocations of live objects. You can specify the gc GET parameter to run GC before taking the heap sample.
- **mutex**: Stack traces of holders of contended mutexes
- **profile**: CPU profile. You can specify the duration in the seconds GET parameter. After you get the profile file, use the go tool pprof command to investigate the profile.
- **threadcreate**: Stack traces that led to the creation of new OS threads
- **trace**: A trace of execution of the current program. You can specify the duration in the seconds GET parameter. After you get the trace file, use the go tool trace command to investigate the trace.

​	注意：对于**trace**类型，需要使用`go tool trace`命令

## 两种访问模式

对profile的分析有两种方式，本地命令行模式以及web可视化模式，本地命令行模式分析heap如下：

```shell
fanqingsong@nvid:~% go tool pprof localhost:PORT/debug/pprof/heap
Fetching profile over HTTP from http://20.20.42.70:9999/debug/pprof/heap
Saved profile in /Users/fanqingsong/pprof/pprof.video_convert.alloc_objects.alloc_space.inuse_objects.inuse_space.007.pb.gz
File: video_convert
Type: inuse_space
Time: Sep 2, 2021 at 11:37am (CST)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top #输入top命令，进行排序
Showing nodes accounting for 512.19kB, 100% of 512.19kB total
      flat  flat%   sum%        cum   cum%
  512.19kB   100%   100%   512.19kB   100%  runtime.malg
         0     0%   100%   512.19kB   100%  runtime.mstart
         0     0%   100%   512.19kB   100%  runtime.newproc.func1
         0     0%   100%   512.19kB   100%  runtime.newproc1
         0     0%   100%   512.19kB   100%  runtime.systemstack
(pprof
```

web可视化模式：**go tool pprof -http=:99**  http://localhost:PORT/debug/pprof/heap

## Heap profile说明

heap profile采集程序内部内存分配及使用情况，分为两大类：**alloc**，**inuse**。其中**alloc**为程序运行以来memory总分配量，其记录的信息为历史信息，一些调用关系此时可能已经执行结束，而**inuse**信息则是当前采集时正在使用的memory分配信息。

```
-inuse_space      Display in-use memory size
-inuse_objects    Display in-use object counts
-alloc_space      Display allocated memory size
-alloc_objects    Display allocated object counts
```



## Cpu profile说明

cpu profile为记录程序执行时各个调用之间的cpu时间开销（默认采集30s），其中如果使用web可视化方式打开(**go tool pprof -http=:99**  http://localhost:PORT/debug/pprof/profile)，可以看见如下Graph关系图：

其中红线越粗代表调用之间花费cpu时间越长，虚线代表间接调用，除了使用Graph显示外，也支持top方式展示和火焰图展示，火焰图展示如下：

其中长条越长，代表调用花费cpu时间越长，点击某个长条可以查看其内部更为具体的花费信息。



## Goroutine profile说明

与heap profile以及cpu profile类型，goroutine profile可以显示程序中groutie的总数以及分别是哪些地方启动了goroutine，如下图所示：

对于goroutine的状况追踪，另一种方式是直接访问： http://localhost:PORT/debug/pprof/goroutine?debug=2，此时浏览器会返回各个goroutine的状态以及trace信息，如下图所示：


比较方便的是可以看到各个goroutine的状态，对于debug时非常方便。

例如：

* `goroutine 449952 [chan receive]`：代表该goroutine处于chan receive状态
* `goroutine 475733 [IO wait, 3 minutes]`：代表该goroutine处于IO wait状态，且等待了**3分钟**

