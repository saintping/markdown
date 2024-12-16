### Flink架构
![flink-architecture.png](https://ping666.com/wp-content/uploads/2024/09/flink-architecture.png "flink-architecture.png")

Flink不仅提供实时流计算，也支持批处理，所谓流批一体。对外提供DataStream和Table API、Flink SQL接口。

![flink-model.png](https://ping666.com/wp-content/uploads/2024/09/flink-model.png "flink-model.png")

### DataStream API
DataStream是Flink的核心数据结构，一般从StreamExecutionEnvironment开始构造数据流。
`flink\flink-streaming-java\src\main\java\org\apache\flink\streaming\api\datastream\DataStream.java`
以下是DataStream的所有方法。
![DataStream-scaled.jpg](https://ping666.com/wp-content/uploads/2024/09/DataStream-scaled.jpg "DataStream-scaled.jpg")

通过DataStream编织完业务逻辑，Flink先将数据流编译成JobGraph，然后就可以将ApplicationMaster提交到Yarn集群计算了。
```java
    private ApplicationReport startAppMaster(
            Configuration configuration,
            String applicationName,
            String yarnClusterEntrypoint,
            JobGraph jobGraph,
            YarnClient yarnClient,
            YarnClientApplication yarnApplication,
            ClusterSpecification clusterSpecification)
            throws Exception {

        // ------------------ Initialize the file systems -------------------------

        org.apache.flink.core.fs.FileSystem.initialize(
                configuration, PluginUtils.createPluginManagerFromRootFolder(configuration));

        final FileSystem fs = FileSystem.get(yarnConfiguration);

        // hard coded check for the GoogleHDFS client because its not overriding the getScheme()
        // method.
        if (!fs.getClass().getSimpleName().equals("GoogleHadoopFileSystem")
                && fs.getScheme().startsWith("file")) {
            LOG.warn(
                    "The file system scheme is '"
                            + fs.getScheme()
                            + "'. This indicates that the "
                            + "specified Hadoop configuration path is wrong and the system is using the default Hadoop configuration values."
                            + "The Flink YARN client needs to store its files in a distributed file system");
        }

        ApplicationSubmissionContext appContext = yarnApplication.getApplicationSubmissionContext();

        final List<Path> providedLibDirs =
                Utils.getQualifiedRemoteSharedPaths(configuration, yarnConfiguration);

        final YarnApplicationFileUploader fileUploader =
                YarnApplicationFileUploader.from(
                        fs,
                        getStagingDir(fs),
                        providedLibDirs,
                        appContext.getApplicationId(),
                        getFileReplication());

        // The files need to be shipped and added to classpath.
        Set<File> systemShipFiles = new HashSet<>(shipFiles.size());
        for (File file : shipFiles) {
            systemShipFiles.add(file.getAbsoluteFile());
        }

        final String logConfigFilePath =
                configuration.getString(YarnConfigOptionsInternal.APPLICATION_LOG_CONFIG_FILE);
        if (logConfigFilePath != null) {
            systemShipFiles.add(new File(logConfigFilePath));
        }

        // Set-up ApplicationSubmissionContext for the application

        final ApplicationId appId = appContext.getApplicationId();

        // ------------------ Add Zookeeper namespace to local flinkConfiguraton ------
        setHAClusterIdIfNotSet(configuration, appId);

        if (HighAvailabilityMode.isHighAvailabilityModeActivated(configuration)) {
            // activate re-execution of failed applications
            appContext.setMaxAppAttempts(
                    configuration.getInteger(
                            YarnConfigOptions.APPLICATION_ATTEMPTS.key(),
                            YarnConfiguration.DEFAULT_RM_AM_MAX_ATTEMPTS));

            activateHighAvailabilitySupport(appContext);
        } else {
            // set number of application retries to 1 in the default case
            appContext.setMaxAppAttempts(
                    configuration.getInteger(YarnConfigOptions.APPLICATION_ATTEMPTS.key(), 1));
        }

        final Set<Path> userJarFiles = new HashSet<>();
        if (jobGraph != null) {
            userJarFiles.addAll(
                    jobGraph.getUserJars().stream()
                            .map(f -> f.toUri())
                            .map(Path::new)
                            .collect(Collectors.toSet()));
        }

        final List<URI> jarUrls =
                ConfigUtils.decodeListFromConfig(configuration, PipelineOptions.JARS, URI::create);
        if (jarUrls != null
                && YarnApplicationClusterEntryPoint.class.getName().equals(yarnClusterEntrypoint)) {
            userJarFiles.addAll(jarUrls.stream().map(Path::new).collect(Collectors.toSet()));
        }

        // only for per job mode
        if (jobGraph != null) {
            for (Map.Entry<String, DistributedCache.DistributedCacheEntry> entry :
                    jobGraph.getUserArtifacts().entrySet()) {
                // only upload local files
                if (!Utils.isRemotePath(entry.getValue().filePath)) {
                    Path localPath = new Path(entry.getValue().filePath);
                    Tuple2<Path, Long> remoteFileInfo =
                            fileUploader.uploadLocalFileToRemote(localPath, entry.getKey());
                    jobGraph.setUserArtifactRemotePath(
                            entry.getKey(), remoteFileInfo.f0.toString());
                }
            }

            jobGraph.writeUserArtifactEntriesToConfiguration();
        }

        if (providedLibDirs == null || providedLibDirs.isEmpty()) {
            addLibFoldersToShipFiles(systemShipFiles);
        }

        // Register all files in provided lib dirs as local resources with public visibility
        // and upload the remaining dependencies as local resources with APPLICATION visibility.
        final List<String> systemClassPaths = fileUploader.registerProvidedLocalResources();
        final List<String> uploadedDependencies =
                fileUploader.registerMultipleLocalResources(
                        systemShipFiles.stream()
                                .map(e -> new Path(e.toURI()))
                                .collect(Collectors.toSet()),
                        Path.CUR_DIR,
                        LocalResourceType.FILE);
        systemClassPaths.addAll(uploadedDependencies);

        // upload and register ship-only files
        // Plugin files only need to be shipped and should not be added to classpath.
        if (providedLibDirs == null || providedLibDirs.isEmpty()) {
            Set<File> shipOnlyFiles = new HashSet<>();
            addPluginsFoldersToShipFiles(shipOnlyFiles);
            fileUploader.registerMultipleLocalResources(
                    shipOnlyFiles.stream()
                            .map(e -> new Path(e.toURI()))
                            .collect(Collectors.toSet()),
                    Path.CUR_DIR,
                    LocalResourceType.FILE);
        }

        if (!shipArchives.isEmpty()) {
            fileUploader.registerMultipleLocalResources(
                    shipArchives.stream().map(e -> new Path(e.toURI())).collect(Collectors.toSet()),
                    Path.CUR_DIR,
                    LocalResourceType.ARCHIVE);
        }

        // Upload and register user jars
        final List<String> userClassPaths =
                fileUploader.registerMultipleLocalResources(
                        userJarFiles,
                        userJarInclusion == YarnConfigOptions.UserJarInclusion.DISABLED
                                ? ConfigConstants.DEFAULT_FLINK_USR_LIB_DIR
                                : Path.CUR_DIR,
                        LocalResourceType.FILE);

        if (userJarInclusion == YarnConfigOptions.UserJarInclusion.ORDER) {
            systemClassPaths.addAll(userClassPaths);
        }

        // normalize classpath by sorting
        Collections.sort(systemClassPaths);
        Collections.sort(userClassPaths);

        // classpath assembler
        StringBuilder classPathBuilder = new StringBuilder();
        if (userJarInclusion == YarnConfigOptions.UserJarInclusion.FIRST) {
            for (String userClassPath : userClassPaths) {
                classPathBuilder.append(userClassPath).append(File.pathSeparator);
            }
        }
        for (String classPath : systemClassPaths) {
            classPathBuilder.append(classPath).append(File.pathSeparator);
        }

        // Setup jar for ApplicationMaster
        final YarnLocalResourceDescriptor localResourceDescFlinkJar =
                fileUploader.uploadFlinkDist(flinkJarPath);
        classPathBuilder
                .append(localResourceDescFlinkJar.getResourceKey())
                .append(File.pathSeparator);

        // write job graph to tmp file and add it to local resource
        // TODO: server use user main method to generate job graph
        if (jobGraph != null) {
            File tmpJobGraphFile = null;
            try {
                tmpJobGraphFile = File.createTempFile(appId.toString(), null);
                try (FileOutputStream output = new FileOutputStream(tmpJobGraphFile);
                        ObjectOutputStream obOutput = new ObjectOutputStream(output)) {
                    obOutput.writeObject(jobGraph);
                }

                final String jobGraphFilename = "job.graph";
                configuration.setString(JOB_GRAPH_FILE_PATH, jobGraphFilename);

                fileUploader.registerSingleLocalResource(
                        jobGraphFilename,
                        new Path(tmpJobGraphFile.toURI()),
                        "",
                        LocalResourceType.FILE,
                        true,
                        false);
                classPathBuilder.append(jobGraphFilename).append(File.pathSeparator);
            } catch (Exception e) {
                LOG.warn("Add job graph to local resource fail.");
                throw e;
            } finally {
                if (tmpJobGraphFile != null && !tmpJobGraphFile.delete()) {
                    LOG.warn("Fail to delete temporary file {}.", tmpJobGraphFile.toPath());
                }
            }
        }

        // Upload the flink configuration
        // write out configuration file
        File tmpConfigurationFile = null;
        try {
            tmpConfigurationFile = File.createTempFile(appId + "-flink-conf.yaml", null);
            BootstrapTools.writeConfiguration(configuration, tmpConfigurationFile);

            String flinkConfigKey = "flink-conf.yaml";
            fileUploader.registerSingleLocalResource(
                    flinkConfigKey,
                    new Path(tmpConfigurationFile.getAbsolutePath()),
                    "",
                    LocalResourceType.FILE,
                    true,
                    true);
            classPathBuilder.append("flink-conf.yaml").append(File.pathSeparator);
        } finally {
            if (tmpConfigurationFile != null && !tmpConfigurationFile.delete()) {
                LOG.warn("Fail to delete temporary file {}.", tmpConfigurationFile.toPath());
            }
        }

        if (userJarInclusion == YarnConfigOptions.UserJarInclusion.LAST) {
            for (String userClassPath : userClassPaths) {
                classPathBuilder.append(userClassPath).append(File.pathSeparator);
            }
        }

        // To support Yarn Secure Integration Test Scenario
        // In Integration test setup, the Yarn containers created by YarnMiniCluster does not have
        // the Yarn site XML
        // and KRB5 configuration files. We are adding these files as container local resources for
        // the container
        // applications (JM/TMs) to have proper secure cluster setup
        Path remoteYarnSiteXmlPath = null;
        if (System.getenv("IN_TESTS") != null) {
            File f = new File(System.getenv("YARN_CONF_DIR"), Utils.YARN_SITE_FILE_NAME);
            LOG.info(
                    "Adding Yarn configuration {} to the AM container local resource bucket",
                    f.getAbsolutePath());
            Path yarnSitePath = new Path(f.getAbsolutePath());
            remoteYarnSiteXmlPath =
                    fileUploader
                            .registerSingleLocalResource(
                                    Utils.YARN_SITE_FILE_NAME,
                                    yarnSitePath,
                                    "",
                                    LocalResourceType.FILE,
                                    false,
                                    false)
                            .getPath();
            if (System.getProperty("java.security.krb5.conf") != null) {
                configuration.set(
                        SecurityOptions.KERBEROS_KRB5_PATH,
                        System.getProperty("java.security.krb5.conf"));
            }
        }

        Path remoteKrb5Path = null;
        boolean hasKrb5 = false;
        String krb5Config = configuration.get(SecurityOptions.KERBEROS_KRB5_PATH);
        if (!StringUtils.isNullOrWhitespaceOnly(krb5Config)) {
            final File krb5 = new File(krb5Config);
            LOG.info(
                    "Adding KRB5 configuration {} to the AM container local resource bucket",
                    krb5.getAbsolutePath());
            final Path krb5ConfPath = new Path(krb5.getAbsolutePath());
            remoteKrb5Path =
                    fileUploader
                            .registerSingleLocalResource(
                                    Utils.KRB5_FILE_NAME,
                                    krb5ConfPath,
                                    "",
                                    LocalResourceType.FILE,
                                    false,
                                    false)
                            .getPath();
            hasKrb5 = true;
        }

        Path remotePathKeytab = null;
        String localizedKeytabPath = null;
        String keytab = configuration.getString(SecurityOptions.KERBEROS_LOGIN_KEYTAB);
        if (keytab != null) {
            boolean localizeKeytab =
                    flinkConfiguration.getBoolean(YarnConfigOptions.SHIP_LOCAL_KEYTAB);
            localizedKeytabPath =
                    flinkConfiguration.getString(YarnConfigOptions.LOCALIZED_KEYTAB_PATH);
            if (localizeKeytab) {
                // Localize the keytab to YARN containers via local resource.
                LOG.info("Adding keytab {} to the AM container local resource bucket", keytab);
                remotePathKeytab =
                        fileUploader
                                .registerSingleLocalResource(
                                        localizedKeytabPath,
                                        new Path(keytab),
                                        "",
                                        LocalResourceType.FILE,
                                        false,
                                        false)
                                .getPath();
            } else {
                // // Assume Keytab is pre-installed in the container.
                localizedKeytabPath =
                        flinkConfiguration.getString(YarnConfigOptions.LOCALIZED_KEYTAB_PATH);
            }
        }

        final JobManagerProcessSpec processSpec =
                JobManagerProcessUtils.processSpecFromConfigWithNewOptionToInterpretLegacyHeap(
                        flinkConfiguration, JobManagerOptions.TOTAL_PROCESS_MEMORY);
        final ContainerLaunchContext amContainer =
                setupApplicationMasterContainer(yarnClusterEntrypoint, hasKrb5, processSpec);

        // setup security tokens
        if (UserGroupInformation.isSecurityEnabled()) {
            // set HDFS delegation tokens when security is enabled
            LOG.info("Adding delegation token to the AM container.");
            List<Path> yarnAccessList =
                    ConfigUtils.decodeListFromConfig(
                            configuration, YarnConfigOptions.YARN_ACCESS, Path::new);
            Utils.setTokensFor(
                    amContainer,
                    ListUtils.union(yarnAccessList, fileUploader.getRemotePaths()),
                    yarnConfiguration);
        }

        amContainer.setLocalResources(fileUploader.getRegisteredLocalResources());
        fileUploader.close();

        // Setup CLASSPATH and environment variables for ApplicationMaster
        final Map<String, String> appMasterEnv = new HashMap<>();
        // set user specified app master environment variables
        appMasterEnv.putAll(
                ConfigurationUtils.getPrefixedKeyValuePairs(
                        ResourceManagerOptions.CONTAINERIZED_MASTER_ENV_PREFIX, configuration));
        // set Flink app class path
        appMasterEnv.put(YarnConfigKeys.ENV_FLINK_CLASSPATH, classPathBuilder.toString());

        // set Flink on YARN internal configuration values
        appMasterEnv.put(YarnConfigKeys.FLINK_DIST_JAR, localResourceDescFlinkJar.toString());
        appMasterEnv.put(YarnConfigKeys.ENV_APP_ID, appId.toString());
        appMasterEnv.put(YarnConfigKeys.ENV_CLIENT_HOME_DIR, fileUploader.getHomeDir().toString());
        appMasterEnv.put(
                YarnConfigKeys.ENV_CLIENT_SHIP_FILES,
                encodeYarnLocalResourceDescriptorListToString(
                        fileUploader.getEnvShipResourceList()));
        appMasterEnv.put(
                YarnConfigKeys.FLINK_YARN_FILES,
                fileUploader.getApplicationDir().toUri().toString());

        // https://github.com/apache/hadoop/blob/trunk/hadoop-yarn-project/hadoop-yarn/hadoop-yarn-site/src/site/markdown/YarnApplicationSecurity.md#identity-on-an-insecure-cluster-hadoop_user_name
        appMasterEnv.put(
                YarnConfigKeys.ENV_HADOOP_USER_NAME,
                UserGroupInformation.getCurrentUser().getUserName());

        if (localizedKeytabPath != null) {
            appMasterEnv.put(YarnConfigKeys.LOCAL_KEYTAB_PATH, localizedKeytabPath);
            String principal = configuration.getString(SecurityOptions.KERBEROS_LOGIN_PRINCIPAL);
            appMasterEnv.put(YarnConfigKeys.KEYTAB_PRINCIPAL, principal);
            if (remotePathKeytab != null) {
                appMasterEnv.put(YarnConfigKeys.REMOTE_KEYTAB_PATH, remotePathKeytab.toString());
            }
        }

        // To support Yarn Secure Integration Test Scenario
        if (remoteYarnSiteXmlPath != null) {
            appMasterEnv.put(
                    YarnConfigKeys.ENV_YARN_SITE_XML_PATH, remoteYarnSiteXmlPath.toString());
        }
        if (remoteKrb5Path != null) {
            appMasterEnv.put(YarnConfigKeys.ENV_KRB5_PATH, remoteKrb5Path.toString());
        }

        // set classpath from YARN configuration
        Utils.setupYarnClassPath(yarnConfiguration, appMasterEnv);

        amContainer.setEnvironment(appMasterEnv);

        // Set up resource type requirements for ApplicationMaster
        Resource capability = Records.newRecord(Resource.class);
        capability.setMemory(clusterSpecification.getMasterMemoryMB());
        capability.setVirtualCores(
                flinkConfiguration.getInteger(YarnConfigOptions.APP_MASTER_VCORES));

        final String customApplicationName = customName != null ? customName : applicationName;

        appContext.setApplicationName(customApplicationName);
        appContext.setApplicationType(applicationType != null ? applicationType : "Apache Flink");
        appContext.setAMContainerSpec(amContainer);
        appContext.setResource(capability);

        // Set priority for application
        int priorityNum = flinkConfiguration.getInteger(YarnConfigOptions.APPLICATION_PRIORITY);
        if (priorityNum >= 0) {
            Priority priority = Priority.newInstance(priorityNum);
            appContext.setPriority(priority);
        }

        if (yarnQueue != null) {
            appContext.setQueue(yarnQueue);
        }

        setApplicationNodeLabel(appContext);

        setApplicationTags(appContext);

        // add a hook to clean up in case deployment fails
        Thread deploymentFailureHook =
                new DeploymentFailureHook(yarnApplication, fileUploader.getApplicationDir());
        Runtime.getRuntime().addShutdownHook(deploymentFailureHook);
        LOG.info("Submitting application master " + appId);
        yarnClient.submitApplication(appContext);

        LOG.info("Waiting for the cluster to be allocated");
        final long startTime = System.currentTimeMillis();
        ApplicationReport report;
        YarnApplicationState lastAppState = YarnApplicationState.NEW;
        loop:
        while (true) {
            try {
                report = yarnClient.getApplicationReport(appId);
            } catch (IOException e) {
                throw new YarnDeploymentException("Failed to deploy the cluster.", e);
            }
            YarnApplicationState appState = report.getYarnApplicationState();
            LOG.debug("Application State: {}", appState);
            switch (appState) {
                case FAILED:
                case KILLED:
                    throw new YarnDeploymentException(
                            "The YARN application unexpectedly switched to state "
                                    + appState
                                    + " during deployment. \n"
                                    + "Diagnostics from YARN: "
                                    + report.getDiagnostics()
                                    + "\n"
                                    + "If log aggregation is enabled on your cluster, use this command to further investigate the issue:\n"
                                    + "yarn logs -applicationId "
                                    + appId);
                    // break ..
                case RUNNING:
                    LOG.info("YARN application has been deployed successfully.");
                    break loop;
                case FINISHED:
                    LOG.info("YARN application has been finished successfully.");
                    break loop;
                default:
                    if (appState != lastAppState) {
                        LOG.info("Deploying cluster, current state " + appState);
                    }
                    if (System.currentTimeMillis() - startTime > 60000) {
                        LOG.info(
                                "Deployment took more than 60 seconds. Please check if the requested resources are available in the YARN cluster");
                    }
            }
            lastAppState = appState;
            Thread.sleep(250);
        }

        // since deployment was successful, remove the hook
        ShutdownHookUtil.removeShutdownHook(deploymentFailureHook, getClass().getSimpleName(), LOG);
        return report;
    }
```

### Table API
Table API比DataStream API更方便一些。以下是实时处理Kafka的数据流。
```java
public class FlinkTemperatureAverage {
    public static void main(String[] args) throws Exception {
        // 设置流执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        // 创建 TableEnvironment
        StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);

        // 定义表结构
        tableEnv.executeSql("CREATE TABLE sensor_readings (timestamp TIMESTAMP(3) METADATA FROM 'timestamp', temperature DOUBLE) WITH ('connector' = 'kafka', 'topic' = 'input_topic', 'properties.bootstrap.servers' = 'localhost:9092', 'format' = 'json')");

        // 创建输出表
        tableEnv.executeSql("CREATE TABLE temperature_averages (window_start TIMESTAMP(3), window_end TIMESTAMP(3), avg_temperature DOUBLE) WITH ('connector' = 'kafka', 'topic' = 'output_topic', 'properties.bootstrap.servers' = 'localhost:9092', 'format' = 'json')");

        // 实时计算每分钟内的平均温度
        tableEnv.executeSql("INSERT INTO temperature_averages SELECT TUMBLE_START(timestamp, INTERVAL '1' MINUTE) AS window_start, TUMBLE_END(timestamp, INTERVAL '1' MINUTE) AS window_end, AVG(temperature) AS avg_temperature FROM sensor_readings GROUP BY TUMBLE(timestamp, INTERVAL '1' MINUTE)");

        // 启动 Flink 任务
        env.execute("Flink Temperature Average Calculation");
    }
}
```

### 窗口
就像Hive SQL中的开窗一样，Flink对于聚合分析也需要开启窗口。

- 滚动窗口（Tumbling Windows）
  固定大小的、不重叠的窗口。例如，每5分钟一个窗口。
- 滑动窗口（Sliding Windows）
  固定大小的、但可以重叠的窗口。例如，每1分钟滑动一次，每次窗口大小为5分钟。
- 会话窗口（Session Windows）
  基于活动时间的窗口，即当数据在一定时间间隔内没有到达时，窗口会关闭。这种窗口适用于处理活动间隔的数据，如用户会话。
- 全局窗口（Global Windows）
  一个全局窗口会包含流中的所有元素，并且通常需要一个触发器（Trigger）来定义何时应该对这个窗口的元素进行计算。
- 计数窗口（Count Windows）
  基于元素数量的窗口，当窗口内的元素数量达到指定值时，就会触发计算。

下面是各种窗口之间的区别
![flink-window.webp](https://ping666.com/wp-content/uploads/2024/09/flink-window.webp "flink-window.webp")

### 数据源
Flink支持的数据源有：文件系统、Kafka、RabbitMQ、JDBC数据库、Socket。
还支持自定义数据源，只要实现SourceFunction和SinkFunction接口就可以了。

### Flink CDC
Flink CDC（Flink Change Data Capture）基于Flink框架，实现了对数据源变更数据的捕捉和处理。
单独的开源项目[https://github.com/apache/flink-cdc](https://github.com/apache/flink-cdc "https://github.com/apache/flink-cdc")

Flink CDC内置了CDC引擎debezium，通过监听binlog等使得捕捉数据的能力比基于SourceFunction的查询方案更加高效。debezium是可以作为服务独立部署的，只是Flink CDC将其作为插件内置进来了。
