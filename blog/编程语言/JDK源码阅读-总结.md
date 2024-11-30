### Java调试
JVM定义了一整套规范的协议和接口用于程序调试，JPDA（ava平台调试器架构）。这也是Java相比其他语言的一大优势。甚至，可以使用JDI接口自己写调试器。

- VisualVM
  口号是All-in-One Java Troubleshooting Tool。在开发&测试环境确实非常给力。可以本地启动，也可以attach到远程，实时观看进程数据。
![visualvm-screenshot-20.png](https://ping666.com/wp-content/uploads/2024/09/visualvm-screenshot-20.png "visualvm-screenshot-20.png")

- JFR
  VisualVM虽然很无敌，但是因为非常影响服务的吞吐量，不太适合在生产环境直接使用。JFR（Java Flight Recorder）可以随时抽样，所以比较适合生产环境。
```bash
jcmd 31113 VM.unlock_commercial_features
jcmd 31113 JFR.start duration=100s filename=flight.jfr
jcmd 31113 JFR.check
jcmd 31113 JFR.stop
jcmd 31113 JFR.dump
```
   以上31113是进程ID，duration是跟踪时间，太长了flight.jfr文件会比较大。
JFR的界面程序从Java8开始就不跟着JDK一起提供了。可以从这里下载[https://adoptium.net/zh-CN/jmc](https://adoptium.net/zh-CN/jmc/ "https://adoptium.net/zh-CN/jmc")
正常情况，跟着红点提示进一步查看就可以了。
![jfr.png](https://ping666.com/wp-content/uploads/2024/09/jfr.png "jfr.png")

- Arthas
  阿里开源的All in One诊断工具，集成了JFR和async profile火焰图等丰富工具。

### JVM总览
最后，以一张图结束本系列文章。

![hostspot11.png](https://ping666-blog.oss-cn-beijing.aliyuncs.com/image/hostspot11.png "hostspot11.png")
