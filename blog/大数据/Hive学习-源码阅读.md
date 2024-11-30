### Hive架构
Hive是基于Hadoop的，直接使用了HDFS和YARN。在MapReduce的基础上为用户提供SQL入口，大大降低了分布式计算的门槛。SQL计算的前提是数据的结构化，所以Hive将结构化数据的元数据存储在MetaStore。将SQL编译成MapReduce任务后放到Hadoop集群上执行。

Hive架构如下图
![hive-architecture.png](https://ping666.com/wp-content/uploads/2024/09/hive-architecture.png "hive-architecture.png")

### MetaStore
MetaStore将HDFS文件对应成关系型数据库里的表，将结构化数据的字段对成表的字段，将这些元数据以Key-Value格式存储内置的derby数据库里，也支持外接到HBase，Mysql等关系数据库。一般生产环境外接Mysql的比较多。

下面是通过表名从MetaStore查询字段列表的代码。
`hive\metastore\src\java\org\apache\hadoop\hive\metastore\HiveMetaStore.java`
```java
    public List<FieldSchema> get_fields_with_environment_context(String db, String tableName,
        final EnvironmentContext envContext)
        throws MetaException, UnknownTableException, UnknownDBException {
      startFunction("get_fields_with_environment_context", ": db=" + db + "tbl=" + tableName);
      String[] names = tableName.split("\\.");
      String base_table_name = names[0];

      Table tbl;
      List<FieldSchema> ret = null;
      Exception ex = null;
      ClassLoader orgHiveLoader = null;
      Configuration curConf = hiveConf;
      try {
        try {
          tbl = get_table_core(db, base_table_name);
        } catch (NoSuchObjectException e) {
          throw new UnknownTableException(e.getMessage());
        }
        if (null == tbl.getSd().getSerdeInfo().getSerializationLib() ||
          hiveConf.getStringCollection(ConfVars.SERDESUSINGMETASTOREFORSCHEMA.varname).contains
          (tbl.getSd().getSerdeInfo().getSerializationLib())) {
          ret = tbl.getSd().getCols();
        } else {
          try {
            if (envContext != null) {
              String addedJars = envContext.getProperties().get("hive.added.jars.path");
              if (org.apache.commons.lang.StringUtils.isNotBlank(addedJars)) {
                //for thread safe
                curConf = getConf();
                orgHiveLoader = curConf.getClassLoader();
                ClassLoader loader = MetaStoreUtils.addToClassPath(orgHiveLoader, org.apache.commons.lang.StringUtils.split(addedJars, ","));
                curConf.setClassLoader(loader);
              }
            }

            Deserializer s = MetaStoreUtils.getDeserializer(curConf, tbl, false);
            ret = MetaStoreUtils.getFieldsFromDeserializer(tableName, s);
          } catch (SerDeException e) {
            StringUtils.stringifyException(e);
            throw new MetaException(e.getMessage());
          }
        }
      } catch (Exception e) {
        ex = e;
        if (e instanceof UnknownDBException) {
          throw (UnknownDBException) e;
        } else if (e instanceof UnknownTableException) {
          throw (UnknownTableException) e;
        } else if (e instanceof MetaException) {
          throw (MetaException) e;
        } else {
          throw newMetaException(e);
        }
      } finally {
        if (orgHiveLoader != null) {
          curConf.setClassLoader(orgHiveLoader);
        }
        endFunction("get_fields_with_environment_context", ret != null, ex, tableName);
      }

      return ret;
    }
```

### SQL语法
Hive SQL只是近可能的靠近SQL标准，和标准SQL差别还是比较大的。
Hive SQL语法定义如下：
`hplsql\src\main\antlr4\org\apache\hive\hplsql\Hplsql.g4`
Hive使用Java编译工具Antlar来解析SQL。Antlar和C/C++体系的Flex + YACC + Bison组合功能类似，实现语法解析。

下面是解析类图：
![BaseSemanticAnalyzer-scaled.jpg](https://ping666.com/wp-content/uploads/2024/09/BaseSemanticAnalyzer-scaled.jpg "BaseSemanticAnalyzer-scaled.jpg")

比如下面这个典型的表关联SQL。
```sql
SELECT
    e.emp_id,
    e.emp_name,
    e.dept_id,
    d.dept_name
FROM
    employees e
LEFT JOIN
    departments d
ON
    e.dept_id = d.dept_id;
```

### Hive SQL编译
Hive执行SQL的核心代码在
`hive\ql\src\java\org\apache\hadoop\hive\ql\Driver.java`

编译过程主要分以下三步：

- 语法解析得到AST
- 语义分析得到Operator Tree
- 根据元数据库信息生成逻辑执行计划

下面是Operator的类继承关系图
![HiveOperator.jpg](https://ping666.com/wp-content/uploads/2024/09/HiveOperator.jpg "HiveOperator.jpg")

编译过程详细代码如下：
```java
  public int compile(String command, boolean resetTaskIds, boolean deferClose) {
    PerfLogger perfLogger = SessionState.getPerfLogger(true);
    perfLogger.PerfLogBegin(CLASS_NAME, PerfLogger.DRIVER_RUN);
    perfLogger.PerfLogBegin(CLASS_NAME, PerfLogger.COMPILE);
    lDrvState.stateLock.lock();
    try {
      lDrvState.driverState = DriverState.COMPILING;
    } finally {
      lDrvState.stateLock.unlock();
    }

    command = new VariableSubstitution(new HiveVariableSource() {
      @Override
      public Map<String, String> getHiveVariable() {
        return SessionState.get().getHiveVariables();
      }
    }).substitute(conf, command);

    String queryStr = command;

    try {
      // command should be redacted to avoid to logging sensitive data
      queryStr = HookUtils.redactLogString(conf, command);
    } catch (Exception e) {
      LOG.warn("WARNING! Query command could not be redacted." + e);
    }

    if (isInterrupted()) {
      return handleInterruption("at beginning of compilation."); //indicate if need clean resource
    }

    if (ctx != null && ctx.getExplainAnalyze() != AnalyzeState.RUNNING) {
      // close the existing ctx etc before compiling a new query, but does not destroy driver
      closeInProcess(false);
    }

    if (resetTaskIds) {
      TaskFactory.resetId();
    }

    String queryId = conf.getVar(HiveConf.ConfVars.HIVEQUERYID);

    //save some info for webUI for use after plan is freed
    this.queryDisplay.setQueryStr(queryStr);
    this.queryDisplay.setQueryId(queryId);

    LOG.info("Compiling command(queryId=" + queryId + "): " + queryStr);

    SessionState.get().setupQueryCurrentTimestamp();

    // Whether any error occurred during query compilation. Used for query lifetime hook.
    boolean compileError = false;
    try {

      // Initialize the transaction manager.  This must be done before analyze is called.
      final HiveTxnManager txnManager = SessionState.get().initTxnMgr(conf);
      // In case when user Ctrl-C twice to kill Hive CLI JVM, we want to release locks

      // if compile is being called multiple times, clear the old shutdownhook
      ShutdownHookManager.removeShutdownHook(shutdownRunner);
      shutdownRunner = new Runnable() {
        @Override
        public void run() {
          try {
            releaseLocksAndCommitOrRollback(false, txnManager);
          } catch (LockException e) {
            LOG.warn("Exception when releasing locks in ShutdownHook for Driver: " +
                e.getMessage());
          }
        }
      };
      ShutdownHookManager.addShutdownHook(shutdownRunner, SHUTDOWN_HOOK_PRIORITY);

      if (isInterrupted()) {
        return handleInterruption("before parsing and analysing the query");
      }
      if (ctx == null) {
        ctx = new Context(conf);
      }

      ctx.setTryCount(getTryCount());
      ctx.setCmd(command);
      ctx.setHDFSCleanup(true);

      perfLogger.PerfLogBegin(CLASS_NAME, PerfLogger.PARSE);
      ASTNode tree = ParseUtils.parse(command, ctx);
      perfLogger.PerfLogEnd(CLASS_NAME, PerfLogger.PARSE);

      // Trigger query hook before compilation
      queryHooks = loadQueryHooks();
      if (queryHooks != null && !queryHooks.isEmpty()) {
        QueryLifeTimeHookContext qhc = new QueryLifeTimeHookContextImpl();
        qhc.setHiveConf(conf);
        qhc.setCommand(command);

        for (QueryLifeTimeHook hook : queryHooks) {
          hook.beforeCompile(qhc);
        }
      }

      perfLogger.PerfLogBegin(CLASS_NAME, PerfLogger.ANALYZE);
      BaseSemanticAnalyzer sem = SemanticAnalyzerFactory.get(queryState, tree);
      List<HiveSemanticAnalyzerHook> saHooks =
          getHooks(HiveConf.ConfVars.SEMANTIC_ANALYZER_HOOK,
              HiveSemanticAnalyzerHook.class);

      // Flush the metastore cache.  This assures that we don't pick up objects from a previous
      // query running in this same thread.  This has to be done after we get our semantic
      // analyzer (this is when the connection to the metastore is made) but before we analyze,
      // because at that point we need access to the objects.
      Hive.get().getMSC().flushCache();

      // Do semantic analysis and plan generation
      if (saHooks != null && !saHooks.isEmpty()) {
        HiveSemanticAnalyzerHookContext hookCtx = new HiveSemanticAnalyzerHookContextImpl();
        hookCtx.setConf(conf);
        hookCtx.setUserName(userName);
        hookCtx.setIpAddress(SessionState.get().getUserIpAddress());
        hookCtx.setCommand(command);
        hookCtx.setHiveOperation(queryState.getHiveOperation());
        for (HiveSemanticAnalyzerHook hook : saHooks) {
          tree = hook.preAnalyze(hookCtx, tree);
        }
        sem.analyze(tree, ctx);
        hookCtx.update(sem);
        for (HiveSemanticAnalyzerHook hook : saHooks) {
          hook.postAnalyze(hookCtx, sem.getAllRootTasks());
        }
      } else {
        sem.analyze(tree, ctx);
      }
      // Record any ACID compliant FileSinkOperators we saw so we can add our transaction ID to
      // them later.
      acidSinks = sem.getAcidFileSinks();

      LOG.info("Semantic Analysis Completed");

      // validate the plan
      sem.validate();
      acidInQuery = sem.hasAcidInQuery();
      perfLogger.PerfLogEnd(CLASS_NAME, PerfLogger.ANALYZE);

      if (isInterrupted()) {
        return handleInterruption("after analyzing query.");
      }

      // get the output schema
      schema = getSchema(sem, conf);
      plan = new QueryPlan(queryStr, sem, perfLogger.getStartTime(PerfLogger.DRIVER_RUN), queryId,
        queryState.getHiveOperation(), schema);

      conf.setQueryString(queryStr);

      conf.set("mapreduce.workflow.id", "hive_" + queryId);
      conf.set("mapreduce.workflow.name", queryStr);

      // initialize FetchTask right here
      if (plan.getFetchTask() != null) {
        plan.getFetchTask().initialize(queryState, plan, null, ctx.getOpContext());
      }

      //do the authorization check
      if (!sem.skipAuthorization() &&
          HiveConf.getBoolVar(conf, HiveConf.ConfVars.HIVE_AUTHORIZATION_ENABLED)) {

        try {
          perfLogger.PerfLogBegin(CLASS_NAME, PerfLogger.DO_AUTHORIZATION);
          doAuthorization(queryState.getHiveOperation(), sem, command);
        } catch (AuthorizationException authExp) {
          console.printError("Authorization failed:" + authExp.getMessage()
              + ". Use SHOW GRANT to get more details.");
          errorMessage = authExp.getMessage();
          SQLState = "42000";
          return 403;
        } finally {
          perfLogger.PerfLogEnd(CLASS_NAME, PerfLogger.DO_AUTHORIZATION);
        }
      }

      if (conf.getBoolVar(ConfVars.HIVE_LOG_EXPLAIN_OUTPUT)) {
        String explainOutput = getExplainOutput(sem, plan, tree);
        if (explainOutput != null) {
          if (conf.getBoolVar(ConfVars.HIVE_LOG_EXPLAIN_OUTPUT)) {
            LOG.info("EXPLAIN output for queryid " + queryId + " : "
              + explainOutput);
          }
          if (conf.isWebUiQueryInfoCacheEnabled()) {
            queryDisplay.setExplainPlan(explainOutput);
          }
        }
      }
      return 0;
    } catch (Exception e) {
      if (isInterrupted()) {
        return handleInterruption("during query compilation: " + e.getMessage());
      }

      compileError = true;
      ErrorMsg error = ErrorMsg.getErrorMsg(e.getMessage());
      errorMessage = "FAILED: " + e.getClass().getSimpleName();
      if (error != ErrorMsg.GENERIC_ERROR) {
        errorMessage += " [Error "  + error.getErrorCode()  + "]:";
      }

      // HIVE-4889
      if ((e instanceof IllegalArgumentException) && e.getMessage() == null && e.getCause() != null) {
        errorMessage += " " + e.getCause().getMessage();
      } else {
        errorMessage += " " + e.getMessage();
      }

      if (error == ErrorMsg.TXNMGR_NOT_ACID) {
        errorMessage += ". Failed command: " + queryStr;
      }

      SQLState = error.getSQLState();
      downstreamError = e;
      console.printError(errorMessage, "\n"
          + org.apache.hadoop.util.StringUtils.stringifyException(e));
      return error.getErrorCode();//todo: this is bad if returned as cmd shell exit
      // since it exceeds valid range of shell return values
    } finally {
      // Trigger post compilation hook. Note that if the compilation fails here then
      // before/after execution hook will never be executed.
      try {
        if (queryHooks != null && !queryHooks.isEmpty()) {
          QueryLifeTimeHookContext qhc = new QueryLifeTimeHookContextImpl();
          qhc.setHiveConf(conf);
          qhc.setCommand(command);
          for (QueryLifeTimeHook hook : queryHooks) {
            hook.afterCompile(qhc, compileError);
          }
        }
      } catch (Exception e) {
        LOG.warn("Failed when invoking query after-compilation hook.", e);
      }

      double duration = perfLogger.PerfLogEnd(CLASS_NAME, PerfLogger.COMPILE)/1000.00;
      ImmutableMap<String, Long> compileHMSTimings = dumpMetaCallTimingWithoutEx("compilation");
      queryDisplay.setHmsTimings(QueryDisplay.Phase.COMPILATION, compileHMSTimings);

      boolean isInterrupted = isInterrupted();
      if (isInterrupted && !deferClose) {
        closeInProcess(true);
      }
      lDrvState.stateLock.lock();
      try {
        if (isInterrupted) {
          lDrvState.driverState = deferClose ? DriverState.EXECUTING : DriverState.ERROR;
        } else {
          lDrvState.driverState = compileError ? DriverState.ERROR : DriverState.COMPILED;
        }
      } finally {
        lDrvState.stateLock.unlock();
      }

      if (isInterrupted) {
        LOG.info("Compiling command(queryId=" + queryId + ") has been interrupted after " + duration + " seconds");
      } else {
        LOG.info("Completed compiling command(queryId=" + queryId + "); Time taken: " + duration + " seconds");
      }
    }
  }
```

### Hive SQL执行
上面的SQL生成的执行计划如下：
```bash
Explain
STAGE DEPENDENCIES:
  Stage-4 is a root stage
  Stage-3 depends on stages: Stage-4
  Stage-0 depends on stages: Stage-3

STAGE PLANS:
  Stage: Stage-4
    Map Reduce Local Work
      Alias -> Map Local Tables:
        d
          Fetch Operator
            limit: -1
      Alias -> Map Local Operator Tree:
        d
          TableScan
            alias: d
            Statistics: Num rows: 1 Data size: 0 Basic stats: PARTIAL Column stats: NONE
            HashTable Sink Operator
              keys:
                0 dept_id (type: string)
                1 dept_id (type: string)

  Stage: Stage-3
    Map Reduce
      Map Operator Tree:
          TableScan
            alias: e
            Statistics: Num rows: 1 Data size: 0 Basic stats: PARTIAL Column stats: NONE
            Map Join Operator
              condition map:
                   Left Outer Join0 to 1
              keys:
                0 dept_id (type: string)
                1 dept_id (type: string)
              outputColumnNames: _col0, _col1, _col2, _col7
              Statistics: Num rows: 1 Data size: 0 Basic stats: PARTIAL Column stats: NONE
              Select Operator
                expressions: _col0 (type: int), _col1 (type: string), _col2 (type: string), _col7 (type: string)
                outputColumnNames: _col0, _col1, _col2, _col3
                Statistics: Num rows: 1 Data size: 0 Basic stats: PARTIAL Column stats: NONE
                File Output Operator
                  compressed: false
                  Statistics: Num rows: 1 Data size: 0 Basic stats: PARTIAL Column stats: NONE
                  table:
                      input format: org.apache.hadoop.mapred.SequenceFileInputFormat
                      output format: org.apache.hadoop.hive.ql.io.HiveSequenceFileOutputFormat
                      serde: org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe
      Local Work:
        Map Reduce Local Work

  Stage: Stage-0
    Fetch Operator
      limit: -1
      Processor Tree:
        ListSink
```

Hive执行SQL主要分以下三步：

- 优化执行计划
- 将执行计划编排成物理执行任务
- 并发执行任务

以root stage（比如上面执行计划中的Stage-4）为单位并发提交Task到集群执行。

详细代码如下：
```java
  public int execute(boolean deferClose) throws CommandNeedRetryException {
    PerfLogger perfLogger = SessionState.getPerfLogger();
    perfLogger.PerfLogBegin(CLASS_NAME, PerfLogger.DRIVER_EXECUTE);

    boolean noName = StringUtils.isEmpty(conf.get(MRJobConfig.JOB_NAME));
    int maxlen = conf.getIntVar(HiveConf.ConfVars.HIVEJOBNAMELENGTH);
    Metrics metrics = MetricsFactory.getInstance();

    String queryId = conf.getVar(HiveConf.ConfVars.HIVEQUERYID);
    // Get the query string from the conf file as the compileInternal() method might
    // hide sensitive information during query redaction.
    String queryStr = conf.getQueryString();

    lDrvState.stateLock.lock();
    try {
      // if query is not in compiled state, or executing state which is carried over from
      // a combined compile/execute in runInternal, throws the error
      if (lDrvState.driverState != DriverState.COMPILED &&
          lDrvState.driverState != DriverState.EXECUTING) {
        SQLState = "HY008";
        errorMessage = "FAILED: query " + queryStr + " has " +
            (lDrvState.driverState == DriverState.INTERRUPT ? "been cancelled" : "not been compiled.");
        console.printError(errorMessage);
        return 1000;
      } else {
        lDrvState.driverState = DriverState.EXECUTING;
      }
    } finally {
      lDrvState.stateLock.unlock();
    }

    maxthreads = HiveConf.getIntVar(conf, HiveConf.ConfVars.EXECPARALLETHREADNUMBER);

    HookContext hookContext = null;

    // Whether there's any error occurred during query execution. Used for query lifetime hook.
    boolean executionError = false;

    try {
      LOG.info("Executing command(queryId=" + queryId + "): " + queryStr);
      // compile and execute can get called from different threads in case of HS2
      // so clear timing in this thread's Hive object before proceeding.
      Hive.get().clearMetaCallTiming();

      plan.setStarted();

      if (SessionState.get() != null) {
        SessionState.get().getHiveHistory().startQuery(queryStr,
            conf.getVar(HiveConf.ConfVars.HIVEQUERYID));
        SessionState.get().getHiveHistory().logPlanProgress(plan);
      }
      resStream = null;

      SessionState ss = SessionState.get();

      hookContext = new HookContext(plan, queryState, ctx.getPathToCS(), ss.getUserFromAuthenticator(),
          ss.getUserIpAddress(), InetAddress.getLocalHost().getHostAddress(), operationId,
          ss.getSessionId(), Thread.currentThread().getName(), ss.isHiveServerQuery(), perfLogger);
      hookContext.setHookType(HookContext.HookType.PRE_EXEC_HOOK);

      for (Hook peh : getHooks(HiveConf.ConfVars.PREEXECHOOKS)) {
        if (peh instanceof ExecuteWithHookContext) {
          perfLogger.PerfLogBegin(CLASS_NAME, PerfLogger.PRE_HOOK + peh.getClass().getName());

          ((ExecuteWithHookContext) peh).run(hookContext);

          perfLogger.PerfLogEnd(CLASS_NAME, PerfLogger.PRE_HOOK + peh.getClass().getName());
        } else if (peh instanceof PreExecute) {
          perfLogger.PerfLogBegin(CLASS_NAME, PerfLogger.PRE_HOOK + peh.getClass().getName());

          ((PreExecute) peh).run(SessionState.get(), plan.getInputs(), plan.getOutputs(),
              Utils.getUGI());

          perfLogger.PerfLogEnd(CLASS_NAME, PerfLogger.PRE_HOOK + peh.getClass().getName());
        }
      }

      // Trigger query hooks before query execution.
      if (queryHooks != null && !queryHooks.isEmpty()) {
        QueryLifeTimeHookContext qhc = new QueryLifeTimeHookContextImpl();
        qhc.setHiveConf(conf);
        qhc.setCommand(queryStr);
        qhc.setHookContext(hookContext);

        for (QueryLifeTimeHook hook : queryHooks) {
          hook.beforeExecution(qhc);
        }
      }

      setQueryDisplays(plan.getRootTasks());
      int mrJobs = Utilities.getMRTasks(plan.getRootTasks()).size();
      int jobs = mrJobs + Utilities.getTezTasks(plan.getRootTasks()).size()
          + Utilities.getSparkTasks(plan.getRootTasks()).size();
      if (jobs > 0) {
        logMrWarning(mrJobs);
        console.printInfo("Query ID = " + queryId);
        console.printInfo("Total jobs = " + jobs);
      }
      if (SessionState.get() != null) {
        SessionState.get().getHiveHistory().setQueryProperty(queryId, Keys.QUERY_NUM_TASKS,
            String.valueOf(jobs));
        SessionState.get().getHiveHistory().setIdToTableMap(plan.getIdToTableNameMap());
      }
      String jobname = Utilities.abbreviate(queryStr, maxlen - 6);

      // A runtime that launches runnable tasks as separate Threads through
      // TaskRunners
      // As soon as a task isRunnable, it is put in a queue
      // At any time, at most maxthreads tasks can be running
      // The main thread polls the TaskRunners to check if they have finished.

      if (isInterrupted()) {
        return handleInterruption("before running tasks.");
      }
      DriverContext driverCxt = new DriverContext(ctx);
      driverCxt.prepare(plan);

      ctx.setHDFSCleanup(true);
      this.driverCxt = driverCxt; // for canceling the query (should be bound to session?)

      SessionState.get().setMapRedStats(new LinkedHashMap<String, MapRedStats>());
      SessionState.get().setStackTraces(new HashMap<String, List<List<String>>>());
      SessionState.get().setLocalMapRedErrors(new HashMap<String, List<String>>());

      // Add root Tasks to runnable
      for (Task<? extends Serializable> tsk : plan.getRootTasks()) {
        // This should never happen, if it does, it's a bug with the potential to produce
        // incorrect results.
        assert tsk.getParentTasks() == null || tsk.getParentTasks().isEmpty();
        driverCxt.addToRunnable(tsk);

        if (metrics != null) {
          tsk.updateTaskMetrics(metrics);
        }
      }

      perfLogger.PerfLogBegin(CLASS_NAME, PerfLogger.RUN_TASKS);
      // Loop while you either have tasks running, or tasks queued up
      while (driverCxt.isRunning()) {

        // Launch upto maxthreads tasks
        Task<? extends Serializable> task;
        while ((task = driverCxt.getRunnable(maxthreads)) != null) {
          TaskRunner runner = launchTask(task, queryId, noName, jobname, jobs, driverCxt);
          if (!runner.isRunning()) {
            break;
          }
        }

        // poll the Tasks to see which one completed
        TaskRunner tskRun = driverCxt.pollFinished();
        if (tskRun == null) {
          continue;
        }
        hookContext.addCompleteTask(tskRun);
        queryDisplay.setTaskResult(tskRun.getTask().getId(), tskRun.getTaskResult());

        Task<? extends Serializable> tsk = tskRun.getTask();
        TaskResult result = tskRun.getTaskResult();

        int exitVal = result.getExitVal();
        if (isInterrupted()) {
          return handleInterruption("when checking the execution result.");
        }
        if (exitVal != 0) {
          if (tsk.ifRetryCmdWhenFail()) {
            driverCxt.shutdown();
            // in case we decided to run everything in local mode, restore the
            // the jobtracker setting to its initial value
            ctx.restoreOriginalTracker();
            throw new CommandNeedRetryException();
          }
          Task<? extends Serializable> backupTask = tsk.getAndInitBackupTask();
          if (backupTask != null) {
            setErrorMsgAndDetail(exitVal, result.getTaskError(), tsk);
            console.printError(errorMessage);
            errorMessage = "ATTEMPT: Execute BackupTask: " + backupTask.getClass().getName();
            console.printError(errorMessage);

            // add backup task to runnable
            if (DriverContext.isLaunchable(backupTask)) {
              driverCxt.addToRunnable(backupTask);
            }
            continue;

          } else {
            setErrorMsgAndDetail(exitVal, result.getTaskError(), tsk);
            invokeFailureHooks(perfLogger, hookContext,
              errorMessage + Strings.nullToEmpty(tsk.getDiagnosticsMessage()), result.getTaskError());
            SQLState = "08S01";
            console.printError(errorMessage);
            driverCxt.shutdown();
            // in case we decided to run everything in local mode, restore the
            // the jobtracker setting to its initial value
            ctx.restoreOriginalTracker();
            return exitVal;
          }
        }

        driverCxt.finished(tskRun);

        if (SessionState.get() != null) {
          SessionState.get().getHiveHistory().setTaskProperty(queryId, tsk.getId(),
              Keys.TASK_RET_CODE, String.valueOf(exitVal));
          SessionState.get().getHiveHistory().endTask(queryId, tsk);
        }

        if (tsk.getChildTasks() != null) {
          for (Task<? extends Serializable> child : tsk.getChildTasks()) {
            if (DriverContext.isLaunchable(child)) {
              driverCxt.addToRunnable(child);
            }
          }
        }
      }
      perfLogger.PerfLogEnd(CLASS_NAME, PerfLogger.RUN_TASKS);

      // in case we decided to run everything in local mode, restore the
      // the jobtracker setting to its initial value
      ctx.restoreOriginalTracker();

      if (driverCxt.isShutdown()) {
        SQLState = "HY008";
        errorMessage = "FAILED: Operation cancelled";
        invokeFailureHooks(perfLogger, hookContext, errorMessage, null);
        console.printError(errorMessage);
        return 1000;
      }

      // remove incomplete outputs.
      // Some incomplete outputs may be added at the beginning, for eg: for dynamic partitions.
      // remove them
      HashSet<WriteEntity> remOutputs = new LinkedHashSet<WriteEntity>();
      for (WriteEntity output : plan.getOutputs()) {
        if (!output.isComplete()) {
          remOutputs.add(output);
        }
      }

      for (WriteEntity output : remOutputs) {
        plan.getOutputs().remove(output);
      }

      hookContext.setHookType(HookContext.HookType.POST_EXEC_HOOK);
      // Get all the post execution hooks and execute them.
      for (Hook peh : getHooks(HiveConf.ConfVars.POSTEXECHOOKS)) {
        if (peh instanceof ExecuteWithHookContext) {
          perfLogger.PerfLogBegin(CLASS_NAME, PerfLogger.POST_HOOK + peh.getClass().getName());

          ((ExecuteWithHookContext) peh).run(hookContext);

          perfLogger.PerfLogEnd(CLASS_NAME, PerfLogger.POST_HOOK + peh.getClass().getName());
        } else if (peh instanceof PostExecute) {
          perfLogger.PerfLogBegin(CLASS_NAME, PerfLogger.POST_HOOK + peh.getClass().getName());

          ((PostExecute) peh).run(SessionState.get(), plan.getInputs(), plan.getOutputs(),
              (SessionState.get() != null ? SessionState.get().getLineageState().getLineageInfo()
                  : null), Utils.getUGI());

          perfLogger.PerfLogEnd(CLASS_NAME, PerfLogger.POST_HOOK + peh.getClass().getName());
        }
      }

      if (SessionState.get() != null) {
        SessionState.get().getHiveHistory().setQueryProperty(queryId, Keys.QUERY_RET_CODE,
            String.valueOf(0));
        SessionState.get().getHiveHistory().printRowCount(queryId);
      }
      releasePlan(plan);
    } catch (CommandNeedRetryException e) {
      executionError = true;
      throw e;
    } catch (Throwable e) {
      executionError = true;
      if (isInterrupted()) {
        return handleInterruption("during query execution: \n" + e.getMessage());
      }

      ctx.restoreOriginalTracker();
      if (SessionState.get() != null) {
        SessionState.get().getHiveHistory().setQueryProperty(queryId, Keys.QUERY_RET_CODE,
            String.valueOf(12));
      }
      // TODO: do better with handling types of Exception here
      errorMessage = "FAILED: Hive Internal Error: " + Utilities.getNameMessage(e);
      if (hookContext != null) {
        try {
          invokeFailureHooks(perfLogger, hookContext, errorMessage, e);
        } catch (Exception t) {
          LOG.warn("Failed to invoke failure hook", t);
        }
      }
      SQLState = "08S01";
      downstreamError = e;
      console.printError(errorMessage + "\n"
          + org.apache.hadoop.util.StringUtils.stringifyException(e));
      return (12);
    } finally {
      // Trigger query hooks after query completes its execution.
      try {
        if (queryHooks != null && !queryHooks.isEmpty()) {
          QueryLifeTimeHookContext qhc = new QueryLifeTimeHookContextImpl();
          qhc.setHiveConf(conf);
          qhc.setCommand(queryStr);
          qhc.setHookContext(hookContext);

          for (QueryLifeTimeHook hook : queryHooks) {
            hook.afterExecution(qhc, executionError);
          }
        }
      } catch (Exception e) {
        LOG.warn("Failed when invoking query after execution hook", e);
      }

      if (SessionState.get() != null) {
        SessionState.get().getHiveHistory().endQuery(queryId);
      }
      if (noName) {
        conf.set(MRJobConfig.JOB_NAME, "");
      }
      double duration = perfLogger.PerfLogEnd(CLASS_NAME, PerfLogger.DRIVER_EXECUTE)/1000.00;

      ImmutableMap<String, Long> executionHMSTimings = dumpMetaCallTimingWithoutEx("execution");
      queryDisplay.setHmsTimings(QueryDisplay.Phase.EXECUTION, executionHMSTimings);

      Map<String, MapRedStats> stats = SessionState.get().getMapRedStats();
      if (stats != null && !stats.isEmpty()) {
        long totalCpu = 0;
        console.printInfo("MapReduce Jobs Launched: ");
        for (Map.Entry<String, MapRedStats> entry : stats.entrySet()) {
          console.printInfo("Stage-" + entry.getKey() + ": " + entry.getValue());
          totalCpu += entry.getValue().getCpuMSec();
        }
        console.printInfo("Total MapReduce CPU Time Spent: " + Utilities.formatMsecToStr(totalCpu));
      }
      boolean isInterrupted = isInterrupted();
      if (isInterrupted && !deferClose) {
        closeInProcess(true);
      }
      lDrvState.stateLock.lock();
      try {
        if (isInterrupted) {
          if (!deferClose) {
            lDrvState.driverState = DriverState.ERROR;
          }
        } else {
          lDrvState.driverState = executionError ? DriverState.ERROR : DriverState.EXECUTED;
        }
      } finally {
        lDrvState.stateLock.unlock();
      }
      if (isInterrupted) {
        LOG.info("Executing command(queryId=" + queryId + ") has been interrupted after " + duration + " seconds");
      } else {
        LOG.info("Completed executing command(queryId=" + queryId + "); Time taken: " + duration + " seconds");
      }
    }

    if (console != null) {
      console.printInfo("OK");
    }

    return (0);
  }
```
