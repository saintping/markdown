### 前言
Java程序的入口是main函数
`public static void main(String[] args) {}`

启动Java程序有两种方式
```bash
java -cp * com.test.graphql.GraphqlApplication
java -jar graphql-0.0.1-SNAPSHOT.jar
```

打开jar包里的MANIFEST.MF文件，可以看到
```bash
Manifest-Version: 1.0
Start-Class: com.test.graphql.GraphqlApplication
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
Spring-Boot-Version: 2.2.8.RELEASE
Main-Class: org.springframework.boot.loader.JarLauncher
```

-jar方式，只是把从jar包里找到main函数的过程委托给了Main-Class（这里是Spring的jar包格式，对应的org.springframework.boot.loader.JarLauncher）。其他路程和-cp方式没有区别。

### 虚拟机启动过程
java -cp * com.test.graphql.GraphqlApplication
这行命令的背后，都发生了什么呢？

一切要从虚拟机的入口函数main说起。`int (int argc, char **argv)`
main函数只是简单透传调用了一下JLI_Launch。相关代码在`src\java.base\share\native\libjli\java.c`
```cpp
/*
 * Entry point.
 */
JNIEXPORT int JNICALL
JLI_Launch(int argc, char ** argv,              /* main argc, argv */
        int jargc, const char** jargv,          /* java args */
        int appclassc, const char** appclassv,  /* app classpath */
        const char* fullversion,                /* full version defined */
        const char* dotversion,                 /* UNUSED dot version defined */
        const char* pname,                      /* program name */
        const char* lname,                      /* launcher name */
        jboolean javaargs,                      /* JAVA_ARGS */
        jboolean cpwildcard,                    /* classpath wildcard*/
        jboolean javaw,                         /* windows-only javaw */
        jint ergo                               /* unused */
)
{
    int mode = LM_UNKNOWN;
    char *what = NULL;
    char *main_class = NULL;
    int ret;
    InvocationFunctions ifn;
    jlong start, end;
    char jvmpath[MAXPATHLEN];
    char jrepath[MAXPATHLEN];
    char jvmcfg[MAXPATHLEN];

    _fVersion = fullversion;
    _launcher_name = lname;
    _program_name = pname;
    _is_java_args = javaargs;
    _wc_enabled = cpwildcard;

    InitLauncher(javaw);
    DumpState();
    if (JLI_IsTraceLauncher()) {
        int i;
        printf("Java args:\n");
        for (i = 0; i < jargc ; i++) {
            printf("jargv[%d] = %s\n", i, jargv[i]);
        }
        printf("Command line args:\n");
        for (i = 0; i < argc ; i++) {
            printf("argv[%d] = %s\n", i, argv[i]);
        }
        AddOption("-Dsun.java.launcher.diag=true", NULL);
    }

    /*
     * SelectVersion() has several responsibilities:
     *
     *  1) Disallow specification of another JRE.  With 1.9, another
     *     version of the JRE cannot be invoked.
     *  2) Allow for a JRE version to invoke JDK 1.9 or later.  Since
     *     all mJRE directives have been stripped from the request but
     *     the pre 1.9 JRE [ 1.6 thru 1.8 ], it is as if 1.9+ has been
     *     invoked from the command line.
     */
    SelectVersion(argc, argv, &main_class);

    CreateExecutionEnvironment(&argc, &argv,
                               jrepath, sizeof(jrepath),
                               jvmpath, sizeof(jvmpath),
                               jvmcfg,  sizeof(jvmcfg));

    if (!IsJavaArgs()) {
        SetJvmEnvironment(argc,argv);
    }

    ifn.CreateJavaVM = 0;
    ifn.GetDefaultJavaVMInitArgs = 0;

    if (JLI_IsTraceLauncher()) {
        start = CounterGet();
    }

    if (!LoadJavaVM(jvmpath, &ifn)) {
        return(6);
    }

    if (JLI_IsTraceLauncher()) {
        end   = CounterGet();
    }

    JLI_TraceLauncher("%ld micro seconds to LoadJavaVM\n",
             (long)(jint)Counter2Micros(end-start));

    ++argv;
    --argc;

    if (IsJavaArgs()) {
        /* Preprocess wrapper arguments */
        TranslateApplicationArgs(jargc, jargv, &argc, &argv);
        if (!AddApplicationOptions(appclassc, appclassv)) {
            return(1);
        }
    } else {
        /* Set default CLASSPATH */
        char* cpath = getenv("CLASSPATH");
        if (cpath != NULL) {
            SetClassPath(cpath);
        }
    }

    /* Parse command line options; if the return value of
     * ParseArguments is false, the program should exit.
     */
    if (!ParseArguments(&argc, &argv, &mode, &what, &ret, jrepath)) {
        return(ret);
    }

    /* Override class path if -jar flag was specified */
    if (mode == LM_JAR) {
        SetClassPath(what);     /* Override class path */
    }

    /* set the -Dsun.java.command pseudo property */
    SetJavaCommandLineProp(what, argc, argv);

    /* Set the -Dsun.java.launcher pseudo property */
    SetJavaLauncherProp();

    /* set the -Dsun.java.launcher.* platform properties */
    SetJavaLauncherPlatformProps();

    return JVMInit(&ifn, threadStackSize, argc, argv, mode, what, ret);
}
```

这个函数主要做的事情有：检查环境变量，LoadJavaVM（从libjvm.so加载符号JNI_CreateJavaVM/JNI_GetDefaultJavaVMInitArgs/JNI_GetCreatedJavaVMs），处理命令行参数CLASSPATH，JVMInit（新建线程执行JavaMain函数）。

JavaMain函数主要做的事情有：InitializeJVM（调用上面加载的JNI_CreateJavaVM初始化JVM），检查module，LoadMainClass（加载用户的java main函数），CallStaticVoidMethod（执行java main函数），LEAVE（销毁jvm并且退出进程）。

JVM初始化代码主要在
`src\hotspot\share\prims\jni.cpp`
内容包括：SafepointMechanism::initialize（初始化Safepoint），创建虚拟机线程JavaThread，init_globals，initialize_java_lang_classes（初始化Java基础库java.lang.*），CompileBroker::compilation_init_phase1（初始化编译模块），call_initPhase3（初始化类加载器），ClassLoader::compile_the_world(编译所有Java自带类库)

其中init_globals最为复杂，代码如下
```cpp
jint init_globals() {
  HandleMark hm;
  management_init();
  bytecodes_init();
  classLoader_init1();
  compilationPolicy_init();
  codeCache_init();
  VM_Version_init();
  os_init_globals();
  stubRoutines_init1();
  jint status = universe_init();  // dependent on codeCache_init and
                                  // stubRoutines_init1 and metaspace_init.
  if (status != JNI_OK)
    return status;

  gc_barrier_stubs_init();   // depends on universe_init, must be before interpreter_init
  interpreter_init();        // before any methods loaded
  invocationCounter_init();  // before any methods loaded
  accessFlags_init();
  templateTable_init();
  InterfaceSupport_init();
  SharedRuntime::generate_stubs();
  universe2_init();  // dependent on codeCache_init and stubRoutines_init1
  javaClasses_init();// must happen after vtable initialization, before referenceProcessor_init
  referenceProcessor_init();
  jni_handles_init();
#if INCLUDE_VM_STRUCTS
  vmStructs_init();
#endif // INCLUDE_VM_STRUCTS

  vtableStubs_init();
  InlineCacheBuffer_init();
  compilerOracle_init();
  dependencyContext_init();

  if (!compileBroker_init()) {
    return JNI_EINVAL;
  }
  VMRegImpl::set_regName();

  if (!universe_post_init()) {
    return JNI_ERR;
  }
  stubRoutines_init2(); // note: StubRoutines need 2-phase init
  MethodHandles::generate_adapters();

#if INCLUDE_NMT
  // Solaris stack is walkable only after stubRoutines are set up.
  // On Other platforms, the stack is always walkable.
  NMT_stack_walkable = true;
#endif // INCLUDE_NMT

  // All the flags that get adjusted by VM_Version_init and os::init_2
  // have been set so dump the flags now.
  if (PrintFlagsFinal || PrintFlagsRanges) {
    JVMFlag::printFlags(tty, false, PrintFlagsRanges);
  }

  return JNI_OK;
}
```

### 加载main函数
LoadMainClass加载main函数是靠Java层的sun.launcher.LauncherHelper类。调用checkAndLoadMain然后就是Java层的类加载器、双亲委派那些。
```cpp
/*
 * Loads a class and verifies that the main class is present and it is ok to
 * call it for more details refer to the java implementation.
 */
static jclass
LoadMainClass(JNIEnv *env, int mode, char *name)
{
    jmethodID mid;
    jstring str;
    jobject result;
    jlong start, end;
    jclass cls = GetLauncherHelperClass(env);
    NULL_CHECK0(cls);
    if (JLI_IsTraceLauncher()) {
        start = CounterGet();
    }
    NULL_CHECK0(mid = (*env)->GetStaticMethodID(env, cls,
                "checkAndLoadMain",
                "(ZILjava/lang/String;)Ljava/lang/Class;"));

    NULL_CHECK0(str = NewPlatformString(env, name));
    NULL_CHECK0(result = (*env)->CallStaticObjectMethod(env, cls, mid,
                                                        USE_STDERR, mode, str));

    if (JLI_IsTraceLauncher()) {
        end   = CounterGet();
        printf("%ld micro seconds to load main class\n",
               (long)(jint)Counter2Micros(end-start));
        printf("----%s----\n", JLDEBUG_ENV_ENTRY);
    }

    return (jclass)result;
}
```

### 执行main函数

执行main函数对虚拟机来说，就是一个普通的Java函数调用。
```cpp
/*
 * The LoadMainClass not only loads the main class, it will also ensure
 * that the main method's signature is correct, therefore further checking
 * is not required. The main method is invoked here so that extraneous java
 * stacks are not in the application stack trace.
 */
mainID = (*env)->GetStaticMethodID(env, mainClass, "main",
                                   "([Ljava/lang/String;)V");
CHECK_EXCEPTION_NULL_LEAVE(mainID);

/* Invoke main method. */
(*env)->CallStaticVoidMethod(env, mainClass, mainID, mainArgs);
```
