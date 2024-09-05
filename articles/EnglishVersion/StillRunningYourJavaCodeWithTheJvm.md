# 还在用JVM跑你的Java代码吗？太慢了，试试Oracle的GraalVM吧
## 前言
对于Java开发者们来说，几乎每天都在和JVM打交道，然而JVM即将过时了。那些对新技术保持敏锐洞察力的开发者，可能已经在生产环境中部署GraalVM生成的二进制程序了，小伙伴们，你们已经用起来了吗？

## 一、GraalVM有何魅力？

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZnruxGibYb4DU619v2PpTGJvSbg0wIrcrSQHTojnzpZ1V1oibTDloPGq1VhibsmqaBDSopmNleOyxdAA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

GraalVM是由**Oracle**官方大力发展和想要推广的**下一代高性能多语言虚拟机**，目前很多框架已支持 GraalVM ，比如Spring 。
GraalVM 的核心优势在于其**Native Image**技术，它能够将Java代码直接编译成**独立的二进制可执行文件**，通过使用即时编译器（JIT）和提前编译器（Ahead-Of-Time, AOT）优化代码执行提高性能，具有以下高级特性：
- 低CPU和内存使用率：GraalVM 具有高效的垃圾回收机制，可以减少内存使用并提高应用性能。

- 多语言支持：GraalVM可以运行JavaScript、Python、Ruby等多种其他语言，从而扩展了其应用范围。

- 极高的安全性：提供了更高级别的安全性，可以更好地保护应用程序免受攻击和数据泄露等安全威胁。

- 快速启动和预热：GraalVM能够实现快速的应用程序启动，并且无需预热即可获得最佳性能。

- 工具链：提供了一套工具，如调试器、性能分析器等，帮助开发者开发和优化应用。

了解完GraalVM的特性，很多小伙伴会说这东西看起来不错，但用于生产环境是刚需吗？实际情况下能解决我们的哪些痛点？我将在下文中为大家解答。

## 二、Java服务慢的问题能解决吗？

Java开发的小伙伴，是否吐槽过Java慢？这里总结下服务慢的槽点主要三个方面：
- 微服务光启动就得花5-6分钟，虚拟内存占用越来越大，要不要再扩点资源？

- VM参数要不要调一下，怎么调？

- 代码里有没有慢SQL，怎么优化？

上面的启动慢和JVM调参两个问题，开发小伙伴确实痛但又不得不面对，只能期望官方解决这个问题。针对代码中慢SQL问题，目前行业内有一些工具，但大多都是基于经验规则审核，不能有效识别性能问题，无法彻底根治。那有没有彻底根治慢问题的方式呢？

### 1）微服务启动慢和JVM调参问题如何彻底解决？

Orace官方新推出的GraalVM，大家的期望实现了，直指痛点解决上面两个问题，官方为什么要干掉JVM呢？

- JVM作为Java程序的运行环境，光启动就得花一大把时间，在JVM启动完成后，才能执行应用程序本身的启动工作，这就是Java程序启动慢的根因。

- JVM调参也是一门技术活，有门槛，依赖经验。



既然是JVM机制导致慢的问题，那官方就彻底干掉JVM，大家也许好奇，GraalVM是如何解决慢的问题的呢？

GraalVM核心功能是可以在本机直接运行高性能低占用的可执行二进制文件（换言之无需JDK环境即可运行）。与基于 JVM 的Java服务运行相比，经过编译得到的原生可执行文件的在启动速度方面有了很大优化，这一过程消除了对 JVM 或其他运行时环境的依赖，并降低了内存占用，从而使得Java程序能够快速启动。这对于云计算和微服务等需要快速启动和低内存使用的场景来说，是非常有益的。

### 2）SQL慢问题，如何从根源上识别代码SQL性能问题？

近期DBdoctor工具的SQL审核功能发布，开发同学的福音来了。DBdoctor是GraalVM的好搭档，在代码开发阶段就能评估出业务SQL未来上线后的真实性能，并给出优化建议，比如推荐最佳索引，解决业务SQL性能问题导致慢的问题。感兴趣的小伙伴可以看这一篇文章：
> [《数据库索引推荐大PK，DBdoctor和资深DBA的终极较量》](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/articles/DatabaseIndexRecommendedLargePk.md)

服务慢的问题能彻底根治了，开发小伙伴可以行动起来了，GraalVM+DBdoctor可以让你的服务启动运行更加丝滑稳定。


## 三、如何使用GraalVM？

下面我们将通过一个简单的Java Demo来演示一下。

### 1）下载graalvm，并查看当前已安装的组件。

命令查看当前已安装的组件，可以看到native-image是默认安装的组件
```Bash
cd graalvm-jdk-<version>_linux-<architecture>/bin
./gu list
ComponentId              Version             Component name                Stability                     Origin
---------------------------------------------------------------------------------------------------------------------------------
graalvm                  23.0.4              GraalVM Core                  Supported
native-image             23.0.4              Native Image                  Early adopter
```
### 2）准备demo代码（maven工程）
这里使用了 native-maven-plugin maven插件，目的是简化编译的流程。当然也可以不使用插件，通过native-image命令直接进行编译。

- a）工程目录结构
```
├── NativeImageDemo
 │   ├── pom.xml
 │   └── src
 │       ├── main
 │       │   ├── java
 │       │   │   └── HelloWorld.java
 │       │   └── resources
```
-  b）HelloWorld.java
```
public class HelloWorld {
    static class Greeter {
        static {
            System.out.println("Greeter is getting ready!");
        }
        public static void greet() {
            System.out.println("Hello, World!");
        }
    }
    public static void main(String[] args) {
        Greeter.greet();
    }
}
```
- c）pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>org.example</groupId>
    <artifactId>NativeImageDemo</artifactId>
    <version>1.0-SNAPSHOT</version>
    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
    </properties>
    <build>
        <plugins>
            <plugin>
                <groupId>org.graalvm.buildtools</groupId>
                <artifactId>native-maven-plugin</artifactId>
                <version>0.10.2</version>
                <configuration>
                    <imageName>${project.artifactId}</imageName>
                    <mainClass>HelloWorld</mainClass>
                    <skipNativeTests>true</skipNativeTests>
                    <buildArgs>
                        <buildArg>-H:+ReportExceptionStackTraces</buildArg>
                    </buildArgs>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```
### 3）编译执行二进制程序

- a）编译

cd 到项目根目录执行以下命令，等待片刻命令行输出 BUILD SUCCESS 说明编译成功。
```Bash
mvn -U clean package native:compile
========================================================================================================================
GraalVM Native Image: Generating 'NativeImageDemo' (executable)...
========================================================================================================================
[1/8] Initializing...                                                                                    (4.8s @ 0.19GB)
 C compiler: gcc (redhat, x86_64, 4.8.5)
 Garbage collector: Serial GC (max heap size: 80% of RAM)
[2/8] Performing analysis...  [****]                                                                     (3.9s @ 0.25GB)
   1,856 (59.16%) of  3,137 types reachable
   1,737 (46.34%) of  3,748 fields reachable
   7,717 (35.62%) of 21,663 methods reachable
     640 types,     0 fields, and   283 methods registered for reflection
      49 types,    32 fields, and    48 methods registered for JNI access
       4 native libraries: dl, pthread, rt, z
[3/8] Building universe...                                                                               (1.0s @ 0.38GB)
[4/8] Parsing methods...      [*]                                                                        (1.5s @ 0.33GB)
[5/8] Inlining methods...     [***]                                                                      (0.4s @ 0.38GB)
[6/8] Compiling methods...    [***]                                                                     (11.0s @ 0.51GB)
[7/8] Layouting methods...    [*]                                                                        (1.0s @ 0.45GB)
[8/8] Creating image...       [*]                                                                        (1.3s @ 0.52GB)
   2.75MB (43.13%) for code area:     3,486 compilation units
   3.46MB (54.34%) for image heap:   48,919 objects and 1 resources
 165.42kB ( 2.53%) for other data
   6.38MB in total
------------------------------------------------------------------------------------------------------------------------
Top 10 origins of code area:                                Top 10 object types in image heap:
   1.43MB java.base                                          549.55kB byte[] for code metadata
   1.13MB svm.jar (Native Image)                             415.45kB byte[] for java.lang.String
  69.54kB com.oracle.svm.svm_enterprise                      325.83kB java.lang.String
  33.89kB org.graalvm.nativeimage.base                       304.98kB java.lang.Class
  30.23kB org.graalvm.sdk                                    253.66kB byte[] for general heap data
  18.95kB jdk.internal.vm.ci                                 147.78kB java.util.HashMap$Node
  14.10kB jdk.internal.vm.compiler                           111.71kB char[]
   1.17kB jdk.proxy3                                          78.91kB java.lang.Object[]
   1.15kB jdk.proxy1                                          72.50kB com.oracle.svm.core.hub.DynamicHubCompanion
  360.00B jdk.proxy2                                          70.45kB byte[] for reflection metadata
  162.00B for 1 more packages                                441.46kB for 506 more object types
------------------------------------------------------------------------------------------------------------------------
Recommendations:
 G1GC: Use the G1 GC ('--gc=G1') for improved latency and throughput.
 PGO:  Use Profile-Guided Optimizations ('--pgo') for improved throughput.
 HEAP: Set max heap for improved and more predictable memory usage.
 CPU:  Enable more CPU features with '-march=native' for improved performance.
 QBM:  Use the quick build mode ('-Ob') to speed up builds during development.
------------------------------------------------------------------------------------------------------------------------
                        0.8s (3.0% of total time) in 99 GCs | Peak RSS: 1.18GB | CPU load: 15.45
------------------------------------------------------------------------------------------------------------------------
Produced artifacts:
 /usr/local/NativeImageDemo/target/NativeImageDemo (executable)
========================================================================================================================
Finished generating 'NativeImageDemo' in 25.6s.
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  28.975 s
[INFO] Finished at: 2024-07-21T13:29:33Z
[INFO] --------------------------------------------------------------
```
- b）执行二进制程序

cd到target目录，执行以下命令即可得到结果。可以看到NativeImageDemo文件是一个可执行文件，可以在任何相同架构的linux服务器上执行。
```Bash
./NativeImageDemo
Greeter is getting ready!
Hello, World!
```
## 四、如何上线前发现性能问题？
DBdoctor下载完成后可以零依赖一分钟安装部署服务，SQL审核主要包括两个部分：
- a）SQL规则审核（规范）：

SQL规则审核相当于公司的SQL规范，通过DBdoctor可以快速识别规范问题并指出问题原因。
![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZkBgQqibQaPCZMwpYbVhw2Jc2s68rxfZVtRzFCHMt7piaUgNIYsaQiajUrMQkxEQ76N0dMpia3anJzEoQ/640?wx_fmt=other&from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)

- b）SQL性能审核:

DBdoctor基于外置COST优化器，通过采集真实数据情况进行计算，最终得出所有索引组合的COST消耗排序，推荐COST最小的索引（最优索引）。只需要把SQL贴在SQL审核里进行审核，就能查看到推荐结果。
![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmS7eht6cbb1icSWkyeCtM1Chrx04vC6TcEcymh80RyJquq9Iv95mDW8QGaQISiaybYy0K9PWBM4Haw/640?wx_fmt=other&from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)

## 五、总结

GraalVM Native Image技术为 Java应用带来快速启动和低资源消耗的优势，DBdoctor的SQL审核技术为Java应用带来快速SQL性能审核和无需生产变更就能评估的优势，GraalVM+DBdoctor的配合能助力Java服务提速，欢迎加入我们的技术交流群与我们探讨交流！