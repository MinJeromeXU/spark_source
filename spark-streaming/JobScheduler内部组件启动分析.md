JobScheduler中有两个比较重要的组件JobGenerator和ReceiverTracker,ReceiverTracker用于接受数据，JobGenerator内部通过一个定时器每个batch定时启动一个job,每次启动之前需要将ReceiverTracker接受到的数据进行划分到属于该batch的job。（JobShceduler内部还有个和自身通信有关的EventLoop，后续分析）

先看如何启动这两个组件的：

JobScheduler在创建的时候其构造器会创建JobGenerator并在其start方法中启动，而ReceiverTracker只是在构造器中进行了声明，在start方法中正真创建并启动。

``` scala
private val jobGenerator = new JobGenerator(this)
def start(): Unit = synchronized {
    if (eventLoop != null) return // scheduler has already been started
	//EventLoop用于自身的通信后，在JobGenerator中也有一个后续分析
    eventLoop = new EventLoop[JobSchedulerEvent]("JobScheduler") {
      override protected def onReceive(event: JobSchedulerEvent): Unit = processEvent(event)

      override protected def onError(e: Throwable): Unit = reportError("Error in job scheduler", e)
    }
    eventLoop.start()

    // attach rate controllers of input streams to receive batch completion updates
    for {
      inputDStream <- ssc.graph.getInputStreams
      rateController <- inputDStream.rateController
    } ssc.addStreamingListener(rateController)
	//省略了暂时分析的代码，先分析主流程
    listenerBus.start()
    receiverTracker = new ReceiverTracker(ssc)//方法内部创建了ReceiverTracker
    inputInfoTracker = new InputInfoTracker(ssc)
	
    receiverTracker.start()//启动receiverTracker
    jobGenerator.start()//启动JobGenerator
  }
  
  
  到此JobGenerator和ReceiverTracker启动流程已经完了，后续分析两个组件的具体实现细节之前先分析EventLoop.
  
  **EventLoop的实现：**
  
  	EventLoop是一个抽象类，其内部主要有几个核心的数据结构，一个基于链表的阻塞队列、一个线程以及两个抽象方法onReceive onError。
	
private val eventQueue: BlockingQueue[E] = new LinkedBlockingDeque[E]()

private val eventThread = new Thread(name) {
    setDaemon(true)
    override def run(): Unit = {
      try {
        while (!stopped.get) {
          val event = eventQueue.take()//从队列中拿事件交给onReceive方法处理，子类只需要实现onReceive方法
          try {
            onReceive(event)
          } catch {             
        }
      } catch {
      }
    }
  }

jobScheduler中start方法创建一个匿名的子类（父类是EventLoop），其内部实现了两个抽象的方法，在onReceive内部接收到事件之后将时间交给了processEvent（e）.
processEvent方法对时间进行模式匹配根据不同的事件处理不同的逻辑。事件主要有三种类型事件。
private def processEvent(event: JobSchedulerEvent) {
    try {
      event match {
        case JobStarted(job, startTime) => handleJobStart(job, startTime)
        case JobCompleted(job, completedTime) => handleJobCompletion(job, completedTime)
        case ErrorReported(m, e) => handleError(m, e)
      }
    } catch {
      case e: Throwable =>
        reportError("Error in job scheduler", e)
    }
  }
  
  分析处理时间之前先考虑事件是如何存入到队列eventQueue的呢？通过JobScheduler内部创建的EventLoop的post方法将时间存入队列中的。那什么时候调用eventLoop的post方法呢？
  看JobScheduler的submitJob提交任务方法。主要将job包装成JbbHandler.交给线程池（这个是固定大小的线程池具体可看代码，这里不分析了）
  
  def submitJobSet(jobSet: JobSet) {
    if (jobSet.jobs.isEmpty) {
      logInfo("No jobs added for time " + jobSet.time)
    } else {
      listenerBus.post(StreamingListenerBatchSubmitted(jobSet.toBatchInfo))
      jobSets.put(jobSet.time, jobSet)
      jobSet.jobs.foreach(job => jobExecutor.execute(new JobHandler(job)))
      logInfo("Added jobs for time " + jobSet.time)
    }
  }
  JobHandler的run方法通过eventLoop向eventQueue中存入了事件并启动了job。
  
  var _eventLoop = eventLoop
   if (_eventLoop != null) {
          _eventLoop.post(JobStarted(job, clock.getTimeMillis())).//存入事件
          // Disable checks for existing output directories in jobs launched by the streaming
          // scheduler, since we may need to write output to an existing directory during checkpoint
          // recovery; see SPARK-4835 for more details.
          SparkHadoopWriterUtils.disableOutputSpecValidation.withValue(true) {
            job.run()
          }
          _eventLoop = eventLoop
          if (_eventLoop != null) {
            _eventLoop.post(JobCompleted(job, clock.getTimeMillis()))
          }
		  
这样事件就存入队列了，现在再来看看processEvent方法不同的事件对于不同的方法handleJobStart和handleJobCompletion主要是事件完成和启动之后的处理listenerBus的处理。
对listenerBus的暂时不分析。
  
  
  



