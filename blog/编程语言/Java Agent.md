### 前言
Java Agent在JDK 1.5中引入，支持在程序启动时动态修改代码逻辑。为了确保程序所有者批准使用，需要显示使用命令行参数-javaagent。Java Agent在JDK 1.6中得到了增强，支持运行时修改程序，可以通过参数禁止该行为-XX:+DisableAttachMechanism。

和常见的侵入式代码增强手段（动态代理、CGLIB、AspectJ等）不同，Java Agent是非侵入式的。工作在程序运行时，意味着不需要修改源代码即可修改程序逻辑。这在热更新和线上排查问题时非常有用。

### Java Agent API
Java Agent类修改的过程如下
![java-agent.png](https://ping666.com/wp-content/uploads/2024/11/java-agent.png "java-agent.png")

Java Agent相关的API主要有两类：

- 类转换器ClassFileTransformer
  类转换器很简单，返回类修改后的字节码就可以。
  ```java
  public interface ClassFileTransformer {
      byte[] transform(ClassLoader        loader,
                      String              className,
                      Class<?>            classBeingRedefined,
                      ProtectionDomain    protectionDomain,
                      byte[]              classfileBuffer)
                      throws IllegalClassFormatException;
  }
  ```

- 类转换器的装载
  类转换器的装载相对要复杂一点。一般是在静态方法premain/agentmain中将需要修改的类转换器装载进Instrumentation中，然后将这两个方法打包成特定格式的jar包。
  ```java
  // 程序启动类加载时调用
  public static void premain(String agentArgs, Instrumentation inst);
  // 程序运行被attch时调用
  public static void agentmain(String agentArgs, Instrumentation inst);
  ```
  Instrumentation接口的完整方法列表。
  ```java
  public interface Instrumentation {
      void addTransformer(ClassFileTransformer transformer, boolean canRetransform);
      void addTransformer(ClassFileTransformer transformer);
      boolean removeTransformer(ClassFileTransformer transformer);
      boolean isRetransformClassesSupported();
      void retransformClasses(Class<?>... classes) throws UnmodifiableClassException;
      boolean isRedefineClassesSupported();
      void redefineClasses(ClassDefinition... definitions) throws ClassNotFoundException, UnmodifiableClassException;
      boolean isModifiableClass(Class<?> theClass);
      Class[] getAllLoadedClasses();
      Class[] getInitiatedClasses(ClassLoader loader);
      long getObjectSize(Object objectToSize);
      void appendToBootstrapClassLoaderSearch(JarFile jarfile);
      void appendToSystemClassLoaderSearch(JarFile jarfile);
      boolean isNativeMethodPrefixSupported();
      void setNativeMethodPrefix(ClassFileTransformer transformer, String prefix);
  }
  ```
  jar包文件META-INF/MANIFEST.MF的内容如下（maven打包插件org.apache.maven.plugins.shade.resource.ManifestResourceTransformer）：
  ```java
  Manifest-Version: 1.0
  Premain-Class: org.example.AttachAgent
  Archiver-Version: Plexus Archiver
  Built-By: matthewliu
  Agent-Class: org.example.AttachAgent
  Can-Redefine-Classes: true
  Can-Retransform-Classes: true
  Created-By: Apache Maven 3.8.1
  Build-Jdk: 1.8.0_261
  ```
  重点是Premain-Class/Agent-Class/Can-Redefine-Classes/Can-Retransform-Classes这4个参数。

**注意：**
transform不能随意修改类，有以下基本限制。否则多个Java Agent一起增强时可能会有冲突。

>The retransformation must not add, remove or rename fields or methods, change the signatures of methods, or change inheritance.

### Java Agent样例

- 被测试类
  一个简单的被测试类org.example.Main，每格3秒打印一行hello world。
  ```java
  package org.example;
  public class Main {
      public static void main(String[] args) {
          new Thread(() -> {
              try {
                  while (true) {
                      new Main().sayHello();
                      Thread.sleep(3000);
                  }
              } catch (InterruptedException ignored) {
              }
          }).start();
      }
  
      private void sayHello() {
          System.out.println("hello world in sayHello");
      }
  }
  ```

- 类转换器
  这里以javassist动态修改一个方法为例。直接加载一个新的外部class文件也是可以的（会覆盖其他探针的修改）。
  ```java
  static class MyTransformer implements ClassFileTransformer {
      @Override
      public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
          // 被测试类org.example.Main
          if ("org/example/Main".equals(className)) {
              // javassist支持的常用功能
              // ClassPool.insertClassPath/makeClass/makeInterface/makeAnnotation;
              // CtClass.addField/addConstructor/addMethod/addInterface/writeFile;
              // 通过反射调用新生成类Object person = CtClass.toClass().newInstance();
              try {
                  System.out.println("try to transformer class: " + className);
                  ClassPool classPool = ClassPool.getDefault();
                  // CtClass ctClass = classPool.get("org.example.Main");支持多个Agent兼容其他探针
                  CtClass ctClass = classPool.makeClassIfNew(new ByteArrayInputStream(classfileBuffer));
                  CtMethod ctMethod = ctClass.getDeclaredMethod("sayHello");
                  // 无论是一行还是多行，最好统一用{}封起来
                  // ctMethod.setBody("{System.out.println(\"hello world in transformed sayHello\");}");
                  ctMethod.insertBefore("{System.out.println(\"hello world in insertBefore\");}");
                  ctMethod.insertAfter("{System.out.println(\"hello world in insertAfter\");}");
                  byte[] bytes = ctClass.toBytecode();
                  ctClass.detach();
                  return bytes;
              } catch (Exception e) {
                  System.out.println("Exception in transform: " + e.getMessage());
                  throw new RuntimeException(e);
              }
          }
          return classfileBuffer;
      }
  }
  ```

- 装载类修改器
  ```java
  public class AttachAgent {
      // 程序启动时类加载时调用
      public static void premain(String agentArgs, Instrumentation inst) {
          System.out.println("premain, agentArgs: " + agentArgs);
          inst.addTransformer(new MyTransformer(), true);
      }
      // 程序被attach时调用，注意要调用retransformClasses通知JVM变更
      public static void agentmain(String agentArgs, Instrumentation inst) {
          System.out.println("agentmain, agentArgs: " + agentArgs);
          inst.addTransformer(new MyTransformer(), true);
          Class<?>[] classes = inst.getAllLoadedClasses();
          for (Class<?> cl : classes) {
              if ("org.example.Main".equals(cl.getName())) {
                  System.out.println("retransformClasses: " + cl.getName());
                  try {
                      inst.retransformClasses(cl);
                  } catch (UnmodifiableClassException e) {
                      System.out.println("Exception in retransformClasses: " + e.getMessage());
                      throw new RuntimeException(e);
                  }
              }
          }
      }
  
      // agentmain需要attach到其他进程
      public static void main(String[] args) {
          List<VirtualMachineDescriptor> vms = VirtualMachine.list();
          for (VirtualMachineDescriptor vm : vms) {
              // 选中被测试类的进程
              if ("org.example.Main".equals(vm.displayName())) {
                  try {
                      VirtualMachine virtualMachine = VirtualMachine.attach(vm.id());
                      virtualMachine.loadAgent("E:/code/AttachAgent/target/AttachAgent-1.0-SNAPSHOT.jar");
                      Thread.sleep(10000);
                      virtualMachine.detach();
                  } catch (Exception e) {
                      System.out.println("Exception in attach: " + e.getMessage());
                  }
              }
          }
      }
  }
  ```

### Java Agent运行
将以上类转换相关代码打包成jar包AttachAgent-1.0-SNAPSHOT.jar

- 程序启动时修改
  被测试类启动时增加以下参数即
  `javaagent:E:/code/AttachAgent/target/AttachAgent-1.0-SNAPSHOT.jar=agent-key=value`
  程序运行效果如下：
  ```java
  premain, agentArgs: agent-key=value
  try to transformer class: org/example/Main
  hello world in insertBefore
  hello world in sayHello
  hello world in insertAfter
  hello world in insertBefore
  hello world in sayHello
  hello world in insertAfter
  ```

- 程序运行时修改
  被测试类正常启动（不加javaagent参数），然后运行attch程序AttachAgent.main。
  程序运行效果如下：
  ```java
  hello world in sayHello
  hello world in sayHello
  hello world in sayHello
  agentmain, agentArgs: null
  retransformClasses: org.example.Main
  try to transformer class: org/example/Main
  hello world in insertBefore
  hello world in sayHello
  hello world in insertAfter
  hello world in insertBefore
  hello world in sayHello
  hello world in insertAfter
  ```

**注意：**

- hotspot的VirtualMachine实现在tools.jar中，需要从jdk目录下拷贝一份到工程本地中才能运行。
- 装载后的类转换器，在不需要时记得使用Instrumentation.removeTransformer移除。否则即使attch进程退出了，类的修改效果也会一直存在。

### SkyWalking探针
SkyWalking是一款APM工具(Application Performance Monitoring)，支持监控性能指标和跟踪消息链路。
[https://skywalking.apache.org/docs/skywalking-java/latest/en/setup/service-agent/java-agent/readme](https://skywalking.apache.org/docs/skywalking-java/latest/en/setup/service-agent/java-agent/readme/ "https://skywalking.apache.org/docs/skywalking-java/latest/en/setup/service-agent/java-agent/readme")

- premain插件加载
  SkyWalking的监控埋点就是通过Java Agent来实现的，字节码增强使用的Byte Buddy。在premain中加载插件，组装ClassFileTransformer。代码在`skywalking-java\apm-sniffer\apm-agent\src\main\java\org\apache\skywalking\apm\agent\SkyWalkingAgent.java`
  ```java
  public class SkyWalkingAgent {
      private static ILog LOGGER = LogManager.getLogger(SkyWalkingAgent.class);
  
      /**
       * Main entrance. Use byte-buddy transform to enhance all classes, which define in plugins.
       */
      public static void premain(String agentArgs, Instrumentation instrumentation) throws PluginException {
          final PluginFinder pluginFinder;
          try {
              SnifferConfigInitializer.initializeCoreConfig(agentArgs);
          } catch (Exception e) {
              // try to resolve a new logger, and use the new logger to write the error log here
              LogManager.getLogger(SkyWalkingAgent.class)
                      .error(e, "SkyWalking agent initialized failure. Shutting down.");
              return;
          } finally {
              // refresh logger again after initialization finishes
              LOGGER = LogManager.getLogger(SkyWalkingAgent.class);
          }
  
          if (!Config.Agent.ENABLE) {
              LOGGER.warn("SkyWalking agent is disabled.");
              return;
          }
  
          try {
              pluginFinder = new PluginFinder(new PluginBootstrap().loadPlugins());
          } catch (AgentPackageNotFoundException ape) {
              LOGGER.error(ape, "Locate agent.jar failure. Shutting down.");
              return;
          } catch (Exception e) {
              LOGGER.error(e, "SkyWalking agent initialized failure. Shutting down.");
              return;
          }
  
          try {
              installClassTransformer(instrumentation, pluginFinder);
          } catch (Exception e) {
              LOGGER.error(e, "Skywalking agent installed class transformer failure.");
          }
  
          try {
              ServiceManager.INSTANCE.boot();
          } catch (Exception e) {
              LOGGER.error(e, "Skywalking agent boot failure.");
          }
  
          Runtime.getRuntime()
                 .addShutdownHook(new Thread(ServiceManager.INSTANCE::shutdown, "skywalking service shutdown thread"));
      }
  }
  ```

- 插件接口
  ClassInstanceMethodsEnhancePluginDefine在这里定义插件需要增强的类和方法以及对应的增强切面。
  ```java
  public class InternalHttpClientInstrumentation extends ClassInstanceMethodsEnhancePluginDefine {
  
      private static final String ENHANCE_CLASS_MINIMAL = "org.apache.hc.client5.http.impl.classic.InternalHttpClient";
      private static final String METHOD_NAME = "doExecute";
      private static final String INTERCEPT_CLASS = "org.apache.skywalking.apm.plugin.httpclient.v5.InternalClientDoExecuteInterceptor";
  
      @Override
      public ClassMatch enhanceClass() {
          return MultiClassNameMatch.byMultiClassMatch(ENHANCE_CLASS_MINIMAL);
      }
  
      @Override
      public ConstructorInterceptPoint[] getConstructorsInterceptPoints() {
          return new ConstructorInterceptPoint[0];
      }
  
      @Override
      public InstanceMethodsInterceptPoint[] getInstanceMethodsInterceptPoints() {
          return new InstanceMethodsInterceptPoint[]{
                  new InstanceMethodsInterceptPoint() {
                      @Override
                      public ElementMatcher<MethodDescription> getMethodsMatcher() {
                          return named(METHOD_NAME);
                      }
  
                      @Override
                      public String getMethodsInterceptor() {
                          return INTERCEPT_CLASS;
                      }
  
                      @Override
                      public boolean isOverrideArgs() {
                          return false;
                      }
                  }
          };
      }
  }
  ```

- 插件增强切面
  增强切面里就是采集Trace链路的Span等信息了。
  ```java
  public abstract class HttpClientDoExecuteInterceptor implements InstanceMethodsAroundInterceptor {
      private static final String ERROR_URI = "/_blank";
  
      private static final ILog LOGGER = LogManager.getLogger(HttpClientDoExecuteInterceptor.class);
  
      @Override
      public void beforeMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes,
              MethodInterceptResult result) throws Throwable {
          if (skipIntercept(objInst, method, allArguments, argumentsTypes)) {
              // illegal args, can't trace. ignore.
              return;
          }
          final HttpHost httpHost = getHttpHost(objInst, method, allArguments, argumentsTypes);
          ClassicHttpRequest httpRequest = (ClassicHttpRequest) allArguments[1];
          final ContextCarrier contextCarrier = new ContextCarrier();
  
          String remotePeer = httpHost.getHostName() + ":" + port(httpHost);
  
          String uri = httpRequest.getUri().toString();
          String requestURI = getRequestURI(uri);
          String operationName = requestURI;
          AbstractSpan span = ContextManager.createExitSpan(operationName, contextCarrier, remotePeer);
          if (ERROR_URI.equals(requestURI)) {
              span.errorOccurred();
          }
          span.setComponent(ComponentsDefine.HTTPCLIENT);
          Tags.URL.set(span, buildURL(httpHost, uri));
          Tags.HTTP.METHOD.set(span, httpRequest.getMethod());
          SpanLayer.asHttp(span);
  
          CarrierItem next = contextCarrier.items();
          while (next.hasNext()) {
              next = next.next();
              httpRequest.setHeader(next.getHeadKey(), next.getHeadValue());
          }
      }
      
      @Override
      public Object afterMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes,
              Object ret) throws Throwable {
          if (skipIntercept(objInst, method, allArguments, argumentsTypes)) {
              return ret;
          }
  
          if (ret != null) {
              ClassicHttpResponse response = (ClassicHttpResponse) ret;
  
              int statusCode = response.getCode();
              AbstractSpan span = ContextManager.activeSpan();
              Tags.HTTP_RESPONSE_STATUS_CODE.set(span, statusCode);
              if (statusCode >= 400) {
                  span.errorOccurred();
              }
          }
  
          ContextManager.stopSpan();
          return ret;
      }
  
      @Override
      public void handleMethodException(EnhancedInstance objInst, Method method, Object[] allArguments,
              Class<?>[] argumentsTypes, Throwable t) {
          if (skipIntercept(objInst, method, allArguments, argumentsTypes)) {
              return;
          }
          AbstractSpan activeSpan = ContextManager.activeSpan();
          activeSpan.log(t);
      }
  }
  ```

### 虚拟机层实现
Java Agent是JVMTI工具集的一部分。

- premain
  启动时的修改比较简单。JVM启动时在Management::initialize中会调用startAgent方法。该方法会调用静态函数premain去装载类转换器。代码在`jdk.management.agent\share\classes\jdk\internal\agent\Agent.java`
  ```java
  /**
   * This method is invoked by the VM to start the management agent
   * when -Dcom.sun.management.* is set during startup.
   */
  public static void startAgent() throws Exception {
      String prop = System.getProperty("com.sun.management.agent.class");
  
      // -Dcom.sun.management.agent.class not set so read management
      // properties and start agent
      if (prop == null) {
          // initialize management properties
          Properties props = getManagementProperties();
          if (props != null) {
              startAgent(props);
          }
          return;
      }
  
      // -Dcom.sun.management.agent.class=<agent classname>:<agent args>
      String[] values = prop.split(":");
      if (values.length < 1 || values.length > 2) {
          error(AGENT_CLASS_INVALID, "\"" + prop + "\"");
      }
      String cname = values[0];
      String args = (values.length == 2 ? values[1] : null);
  
      if (cname == null || cname.length() == 0) {
          error(AGENT_CLASS_INVALID, "\"" + prop + "\"");
      }
  
      if (cname != null) {
          try {
              // Instantiate the named class.
              // invoke the premain(String args) method
              Class<?> clz = ClassLoader.getSystemClassLoader().loadClass(cname);
              Method premain = clz.getMethod("premain",
                      new Class<?>[]{String.class});
              premain.invoke(null, /* static */
                      new Object[]{args});
          } catch (ClassNotFoundException ex) {
              error(AGENT_CLASS_NOT_FOUND, "\"" + cname + "\"");
          } catch (NoSuchMethodException ex) {
              error(AGENT_CLASS_PREMAIN_NOT_FOUND, "\"" + cname + "\"");
          } catch (SecurityException ex) {
              error(AGENT_CLASS_ACCESS_DENIED);
          } catch (Exception ex) {
              String msg = (ex.getCause() == null
                      ? ex.getMessage()
                      : ex.getCause().getMessage());
              error(AGENT_CLASS_FAILED, msg);
          }
      }
  }
  ```

- agentmain
  JPLISAgent主要是初始化java.lang.instrument.Instrumentation，相关代码在`java.instrument\share\native\libinstrument\InvocationAdapter.c`
  ```c
  /*
   *  This will be called once each time a tool attaches to the VM and loads
   *  the JPLIS library.
   */
  JNIEXPORT jint JNICALL
  DEF_Agent_OnAttach(JavaVM* vm, char *args, void * reserved) {
      JPLISInitializationError initerror  = JPLIS_INIT_ERROR_NONE;
      jint                     result     = JNI_OK;
      JPLISAgent *             agent      = NULL;
      JNIEnv *                 jni_env    = NULL;
  
      /*
       * Need JNIEnv - guaranteed to be called from thread that is already
       * attached to VM
       */
      result = (*vm)->GetEnv(vm, (void**)&jni_env, JNI_VERSION_1_2);
      jplis_assert(result==JNI_OK);
  
      initerror = createNewJPLISAgent(vm, &agent);
      if ( initerror == JPLIS_INIT_ERROR_NONE ) {
          int             oldLen, newLen;
          char *          jarfile;
          char *          options;
          jarAttribute*   attributes;
          char *          agentClass;
          char *          bootClassPath;
          jboolean        success;
  
          /*
           * Parse <jarfile>[=options] into jarfile and options
           */
          if (parseArgumentTail(args, &jarfile, &options) != 0) {
              return JNI_ENOMEM;
          }
  
          /*
           * Open the JAR file and parse the manifest
           */
          attributes = readAttributes( jarfile );
          if (attributes == NULL) {
              fprintf(stderr, "Error opening zip file or JAR manifest missing: %s\n", jarfile);
              free(jarfile);
              if (options != NULL) free(options);
              return AGENT_ERROR_BADJAR;
          }
  
          agentClass = getAttribute(attributes, "Agent-Class");
          if (agentClass == NULL) {
              fprintf(stderr, "Failed to find Agent-Class manifest attribute from %s\n",
                  jarfile);
              free(jarfile);
              if (options != NULL) free(options);
              freeAttributes(attributes);
              return AGENT_ERROR_BADJAR;
          }
  
          /*
           * Add the jarfile to the system class path
           */
          if (appendClassPath(agent, jarfile)) {
              fprintf(stderr, "Unable to add %s to system class path "
                  "- not supported by system class loader or configuration error!\n",
                  jarfile);
              free(jarfile);
              if (options != NULL) free(options);
              freeAttributes(attributes);
              return AGENT_ERROR_NOTONCP;
          }
  
          /*
           * The value of the Agent-Class attribute becomes the agent
           * class name. The manifest is in UTF8 so need to convert to
           * modified UTF8 (see JNI spec).
           */
          oldLen = (int)strlen(agentClass);
          newLen = modifiedUtf8LengthOfUtf8(agentClass, oldLen);
          if (newLen == oldLen) {
              agentClass = strdup(agentClass);
          } else {
              char* str = (char*)malloc( newLen+1 );
              if (str != NULL) {
                  convertUtf8ToModifiedUtf8(agentClass, oldLen, str, newLen);
              }
              agentClass = str;
          }
          if (agentClass == NULL) {
              free(jarfile);
              if (options != NULL) free(options);
              freeAttributes(attributes);
              return JNI_ENOMEM;
          }
  
          /*
           * If the Boot-Class-Path attribute is specified then we process
           * each URL - in the live phase only JAR files will be added.
           */
          bootClassPath = getAttribute(attributes, "Boot-Class-Path");
          if (bootClassPath != NULL) {
              appendBootClassPath(agent, jarfile, bootClassPath);
          }
  
          /*
           * Convert JAR attributes into agent capabilities
           */
          convertCapabilityAttributes(attributes, agent);
  
          /*
           * Create the java.lang.instrument.Instrumentation instance
           */
          success = createInstrumentationImpl(jni_env, agent);
          jplis_assert(success);
  
          /*
           * Setup ClassFileLoadHook handler.
           */
          if (success) {
              success = setLivePhaseEventHandlers(agent);
              jplis_assert(success);
          }
  
          /*
           * Start the agent
           */
          if (success) {
              success = startJavaAgent(agent,
                                       jni_env,
                                       agentClass,
                                       options,
                                       agent->mAgentmainCaller);
          }
  
          if (!success) {
              fprintf(stderr, "Agent failed to start!\n");
              result = AGENT_ERROR_STARTFAIL;
          }
  
          /*
           * Clean-up
           */
          free(jarfile);
          if (options != NULL) free(options);
          free(agentClass);
          freeAttributes(attributes);
      }
  
      return result;
  }
  ```

- Instrumentation.retransformClasses
  Instrumentation接口的实现类是sun.instrument.InstrumentationImpl，通过JPLISAgent.c里的retransformClasses最终会调用到jvmtiEnv.cpp里的RetransformClasses。在这里会触发JVM虚拟机线程在STW状态下去执行VM_RedefineClasses。
  ```cpp
  jvmtiError JvmtiEnv::RetransformClasses(jint class_count, const jclass* classes) {
  //TODO: add locking
  
    int index;
    JavaThread* current_thread = JavaThread::current();
    ResourceMark rm(current_thread);
  
    jvmtiClassDefinition* class_definitions =
                              NEW_RESOURCE_ARRAY(jvmtiClassDefinition, class_count);
    NULL_CHECK(class_definitions, JVMTI_ERROR_OUT_OF_MEMORY);
  
    for (index = 0; index < class_count; index++) {
      HandleMark hm(current_thread);
  
      jclass jcls = classes[index];
      oop k_mirror = JNIHandles::resolve_external_guard(jcls);
      if (k_mirror == NULL) {
        return JVMTI_ERROR_INVALID_CLASS;
      }
      if (!k_mirror->is_a(SystemDictionary::Class_klass())) {
        return JVMTI_ERROR_INVALID_CLASS;
      }
  
      if (!VM_RedefineClasses::is_modifiable_class(k_mirror)) {
        return JVMTI_ERROR_UNMODIFIABLE_CLASS;
      }
  
      Klass* klass = java_lang_Class::as_Klass(k_mirror);
  
      jint status = klass->jvmti_class_status();
      if (status & (JVMTI_CLASS_STATUS_ERROR)) {
        return JVMTI_ERROR_INVALID_CLASS;
      }
  
      InstanceKlass* ik = InstanceKlass::cast(klass);
      if (ik->get_cached_class_file_bytes() == NULL) {
        // Not cached, we need to reconstitute the class file from the
        // VM representation. We don't attach the reconstituted class
        // bytes to the InstanceKlass here because they have not been
        // validated and we're not at a safepoint.
        JvmtiClassFileReconstituter reconstituter(ik);
        if (reconstituter.get_error() != JVMTI_ERROR_NONE) {
          return reconstituter.get_error();
        }
  
        class_definitions[index].class_byte_count = (jint)reconstituter.class_file_size();
        class_definitions[index].class_bytes      = (unsigned char*)
                                                         reconstituter.class_file_bytes();
      } else {
        // it is cached, get it from the cache
        class_definitions[index].class_byte_count = ik->get_cached_class_file_len();
        class_definitions[index].class_bytes      = ik->get_cached_class_file_bytes();
      }
      class_definitions[index].klass              = jcls;
    }
    VM_RedefineClasses op(class_count, class_definitions, jvmti_class_load_kind_retransform);
    VMThread::execute(&op);
    return (op.check_error());
  } /* end RetransformClasses */
  ```