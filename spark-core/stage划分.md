DAG stage划分 task生成 task提交执行 task结果的处理；
按照案列分析：
    以action操作的.count为例。
        action操作的执行都会调用sc.runjob。经过方法的重载。主要是构建一个results数组。数组大小和分区大小相等并提供的一个                       resulthandler={（index.res） =>results[index] = res}最终返回这个results。runjob的执行先通过DAGSchedule的runjob方法，在runjob方法内部调用submitJob方法返回一个JobWaiter和消息循环体发送提交job消息.
1、(JobWaiter内部有我们的resultHandler)。在任务执行成功之后会调用jobwaiter的taskSuccessed方法，在该方法内部调用resulthandler将结果写入到我们最终的数组最终返回。
2、消息循环体对提交job的处理。这个就是涉及到了stage的划分，task的提交，执行返回结果的处理最终回到了jobwaiter中。
    **stage的划分：**
        首先根据finalRdd创建一个finalStage.然后调用submit方法提交finalStage。在submit内部实现判断stage有没有没有提交的父stage，遍历stage的rdd，找到没有提交的父stage.主要是rdd依赖的宽依赖和窄依赖区分。递归提交父stage.也就是说最先提交的就是算子的最前面的那个stage。提交stage之后就会提交这个stage的task的方法submitMissingTasks方法提交任务到executor中。
    主要看看SubmitMissingTasks,在这个方法末尾会调用submitWaitingChildStage（提交子stage），那他是如何提交任务的呢。
现将stage序列化之后进行广播到每个worker中。根据stage的不同（ShuffleMapStage   ResultStage）产生不同的task(ShuffleMapTask，ResultTask)。最后提交任务。（正在这中间有一个数据本地化的选择）。Task产生之后通过TaskSchedulerImpl提交task(submitTasks)在该方法内部创建一个TaskSetManager,并加入到了调度池中。然后就是给自己发送消息资源分配（判断有没有新加入的executor（有的话需要重新计算数据本地化），以及黑名单，和数据本地化级别来计算task（task会被封装成taskDescription）进入到哪一个Executor），然后给Executor发送序列化之后的任务。
提交task给CoarseGrainedExecutorBackend然后调用Executor的launchTask方法启动任务，内部执行时将任务包装成一个taskrunner执行，将执行后的结果根据结果集的大小封装成DirectResult还是IndirectResult。
    大小区分主要有最大结果集大小（1G）(超过了就直接丢弃)，然后是是否超过了直接结果集大小（128M）（超过了就使用IndirectResult没有就用directresult）
    其中IndirectResult和directresult区别（IndirectResult返回的blockId以及大小，directresult直接返回数据。）
返回结果主要CoarseCoarseGrainedExecutorBackend给CoarseGrainedScheduleBackend发送statusupdates.
Driver端根据结果类型判断是否需要从远程获取数据（对应IndirectResult需要根据blockId获取数据），最后会调用DAGSchedule的taskEnded方法。
最后会调用DAG的handleTaskCompletion方法，根据是否是ResultStage还是SHuffleMapStage对结果做不同的处理。
    ResultStage会调用jobwaiter（也就是job.listener）的taskSuccessed方法。
    ShuffleMapStage会交给MapOutputTrack.供一下一个stage使用，存放完之后还会调度下一个stage运行。
整个流程就分析完。
**Stage划分的具体算法**。
    **方法调用栈：**
    submitStage--->getMissingParentStage-->getOrCreateShuffleMapStage-->getMissingAncestorShuffleDependencies-->getShuffleDenpencies.所有的ShuffleRDD进去的是一个Stack堆栈。
getMissingAncestorShuffleDenpencies拿到所有的ShuffleRDD遍历调用createShuffleMapStage
createShuffleMapStage-->(1、getOrCreateSHuffleMapstage拿到父stage，创建stage=new ShuffleMapStage(参数有一个是父stage).
Stage划分结束之后就提交父stage为kong的那个stage,也就是第一个stage.提交完一个方法之后调用SubmitWaitingChildStage（判断谁的筛选出谁的父stage是本次提交的stage，然后遍历提交这些父stage是本次stage的自stage）
各个方法的解释
    submitStage：提交stage,里面stage是否有父stage不同操作，有酒拿父stage循环递归调用stage,没有就提交任务调用                                                submitMissingTask
    getMissingParentStage:获取父stage，主要是根据rdd的区分，窄依赖还是宽依赖，宽依赖调用getMissingAncestorShuffleDependencies
在getMissingAncestorShuffleDependencies循环递归调用getShuffleDenpencies，
getShuffleDenpencies该方法只能获取到当前rdd的宽依赖rdd,
拿到所有的宽依赖的RDD那么就可以递归的调用createShuffleMapStage创建Stage. 