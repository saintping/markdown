### Yarn架构
![yarn-architecture.png](https://ping666.com/wp-content/uploads/2024/09/yarn-architecture.png "yarn-architecture.png")

一般的分布式计算都是调度中心（这里的ResourceManager）直接负责管理计算任务，在执行节点（这里的NodeManager）上启动任务并且监控它的执行等。这也是Hadoop早期的做法。后面引入了ApplicationMaster专门用来负责应用任务的调度和监控，从而把任务调度和资源管理完全解耦。所以后续的Hive、Spark、Flink等分布式计算框架都能将任务运行在Hadoop Yarn上，只需要写一个自定义的ApplicationMaster。

### ApplicationMaster
Hadoop MapReduce自己的ApplicationMaster是MRAppMaster，代码在
`hadoop\hadoop-mapreduce-project\hadoop-mapreduce-client\hadoop-mapreduce-client-app\src\main\java\org\apache\hadoop\mapreduce\v2\app\MRAppMaster.java`
```java
  protected void serviceStart() throws Exception {

    amInfos = new LinkedList<AMInfo>();
    completedTasksFromPreviousRun = new HashMap<TaskId, TaskInfo>();
    processRecovery();
    cleanUpPreviousJobOutput();

    // Current an AMInfo for the current AM generation.
    AMInfo amInfo =
        MRBuilderUtils.newAMInfo(appAttemptID, startTime, containerID, nmHost,
            nmPort, nmHttpPort);

    // /////////////////// Create the job itself.
    job = createJob(getConfig(), forcedState, shutDownMessage);

    // End of creating the job.

    // Send out an MR AM inited event for all previous AMs.
    for (AMInfo info : amInfos) {
      dispatcher.getEventHandler().handle(
          new JobHistoryEvent(job.getID(), new AMStartedEvent(info
              .getAppAttemptId(), info.getStartTime(), info.getContainerId(),
              info.getNodeManagerHost(), info.getNodeManagerPort(), info
                  .getNodeManagerHttpPort(), appSubmitTime)));
    }

    // Send out an MR AM inited event for this AM.
    dispatcher.getEventHandler().handle(
        new JobHistoryEvent(job.getID(), new AMStartedEvent(amInfo
            .getAppAttemptId(), amInfo.getStartTime(), amInfo.getContainerId(),
            amInfo.getNodeManagerHost(), amInfo.getNodeManagerPort(), amInfo
                .getNodeManagerHttpPort(), this.forcedState == null ? null
                    : this.forcedState.toString(), appSubmitTime)));
    amInfos.add(amInfo);

    // metrics system init is really init & start.
    // It's more test friendly to put it here.
    DefaultMetricsSystem.initialize("MRAppMaster");

    boolean initFailed = false;
    if (!errorHappenedShutDown) {
      // create a job event for job initialization
      JobEvent initJobEvent = new JobEvent(job.getID(), JobEventType.JOB_INIT);
      // Send init to the job (this does NOT trigger job execution)
      // This is a synchronous call, not an event through dispatcher. We want
      // job-init to be done completely here.
      jobEventDispatcher.handle(initJobEvent);

      // If job is still not initialized, an error happened during
      // initialization. Must complete starting all of the services so failure
      // events can be processed.
      initFailed = (((JobImpl)job).getInternalState() != JobStateInternal.INITED);

      // JobImpl's InitTransition is done (call above is synchronous), so the
      // "uber-decision" (MR-1220) has been made.  Query job and switch to
      // ubermode if appropriate (by registering different container-allocator
      // and container-launcher services/event-handlers).

      if (job.isUber()) {
        speculatorEventDispatcher.disableSpeculation();
        LOG.info("MRAppMaster uberizing job " + job.getID()
            + " in local container (\"uber-AM\") on node "
            + nmHost + ":" + nmPort + ".");
      } else {
        // send init to speculator only for non-uber jobs. 
        // This won't yet start as dispatcher isn't started yet.
        dispatcher.getEventHandler().handle(
            new SpeculatorEvent(job.getID(), clock.getTime()));
        LOG.info("MRAppMaster launching normal, non-uberized, multi-container "
            + "job " + job.getID() + ".");
      }
      // Start ClientService here, since it's not initialized if
      // errorHappenedShutDown is true
      clientService.start();
    }
    //start all the components
    super.serviceStart();

    // finally set the job classloader
    MRApps.setClassLoader(jobClassLoader, getConfig());

    if (initFailed) {
      JobEvent initFailedEvent = new JobEvent(job.getID(), JobEventType.JOB_INIT_FAILED);
      jobEventDispatcher.handle(initFailedEvent);
    } else {
      // All components have started, start the job.
      startJobs();
    }
  }
```

### Job启动
Job在Yarn上的启动代码在这里（本地启动见上一篇）
`hadoop\hadoop-mapreduce-project\hadoop-mapreduce-client\hadoop-mapreduce-client-jobclient\src\main\java\org\apache\hadoop\mapred\YARNRunner.java`
```java
  public JobStatus submitJob(JobID jobId, String jobSubmitDir, Credentials ts)
  throws IOException, InterruptedException {
    
    addHistoryToken(ts);

    ApplicationSubmissionContext appContext =
      createApplicationSubmissionContext(conf, jobSubmitDir, ts);

    // Submit to ResourceManager
    try {
      ApplicationId applicationId =
          resMgrDelegate.submitApplication(appContext);

      ApplicationReport appMaster = resMgrDelegate
          .getApplicationReport(applicationId);
      String diagnostics =
          (appMaster == null ?
              "application report is null" : appMaster.getDiagnostics());
      if (appMaster == null
          || appMaster.getYarnApplicationState() == YarnApplicationState.FAILED
          || appMaster.getYarnApplicationState() == YarnApplicationState.KILLED) {
        throw new IOException("Failed to run job : " +
            diagnostics);
      }
      return clientCache.getClient(jobId).getJobStatus(jobId);
    } catch (YarnException e) {
      throw new IOException(e);
    }
  }
```

创建Application，提交到ResourceManager，然后就是等待Job结果。

ResourceManager启动Application的代码在这里
`hadoop\hadoop-yarn-project\hadoop-yarn\hadoop-yarn-server\hadoop-yarn-server-resourcemanager\src\main\java\org\apache\hadoop\yarn\server\resourcemanager\RMAppManager.java`
```java
  protected void submitApplication(
      ApplicationSubmissionContext submissionContext, long submitTime,
      UserGroupInformation userUgi) throws YarnException {
    ApplicationId applicationId = submissionContext.getApplicationId();

    // Passing start time as -1. It will be eventually set in RMAppImpl
    // constructor.
    RMAppImpl application = createAndPopulateNewRMApp(
        submissionContext, submitTime, userUgi, false, -1, null);
    try {
      if (UserGroupInformation.isSecurityEnabled()) {
        this.rmContext.getDelegationTokenRenewer()
            .addApplicationAsync(applicationId,
                BuilderUtils.parseCredentials(submissionContext),
                submissionContext.getCancelTokensWhenComplete(),
                application.getUser(),
                BuilderUtils.parseTokensConf(submissionContext));
      } else {
        // Dispatcher is not yet started at this time, so these START events
        // enqueued should be guaranteed to be first processed when dispatcher
        // gets started.
        this.rmContext.getDispatcher().getEventHandler()
            .handle(new RMAppEvent(applicationId, RMAppEventType.START));
      }
    } catch (Exception e) {
      LOG.warn("Unable to parse credentials for " + applicationId, e);
      // Sending APP_REJECTED is fine, since we assume that the
      // RMApp is in NEW state and thus we haven't yet informed the
      // scheduler about the existence of the application
      this.rmContext.getDispatcher().getEventHandler()
          .handle(new RMAppEvent(applicationId,
              RMAppEventType.APP_REJECTED, e.getMessage()));
      throw RPCUtil.getRemoteException(e);
    }
  }
```
