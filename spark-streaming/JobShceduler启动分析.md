**StreamingContext**
streamingContext是spark streamin计算的入口，其构造器中创建了JobScheduler并通过内部的start方法另起一个线程启动这个JobScheduler。启动JobScheduler之前还有一个验证，验证Graph是否有效以及检查点.

``` scala
private[streaming] val scheduler = new JobScheduler(this)
def start(): Unit = synchronized {
    state match {
      case INITIALIZED =>
        StreamingContext.ACTIVATION_LOCK.synchronized {
          try {.
            ThreadUtils.runInNewThread("streaming-start") {
              sparkContext.setCallSite(startSite.get)
              sparkContext.clearJobGroup()              sparkContext.setLocalProperty(SparkContext.SPARK_JOB_INTERRUPT_ON_CANCEL, "false")              savedProperties.set(SerializationUtils.clone(sparkContext.localProperties.get()))
              scheduler.start()//新起一个线程启动JobScheduler
            }
            state = StreamingContextState.ACTIVE
            scheduler.listenerBus.post(
              StreamingListenerStreamingStarted(System.currentTimeMillis()))
          } catch {
            ....//省略部分代码
          }
          StreamingContext.setActiveContext(this)
        }     
    }
  }
  
  
Streaming Context主要执行逻辑发生在JobScheduler中，后续分析JobScheduler以及内部的核心组件


