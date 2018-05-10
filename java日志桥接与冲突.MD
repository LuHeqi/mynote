### 桥接
跟 JCL 一样，SLF4J 也是只提供 log 接口，具体的实现是在打包应用程序时所放入的绑定器（名字为 slf4j-XXX-version.jar）来决定，
XXX 可以是 log4j12, jdk14, jcl, nop 等，他们实现了跟具体日志工具（比如 log4j）的绑定及代理工作。
举个例子：如果一个程序希望用 log4j 日志工具，那么程序只需针对 slf4j-api 接口编程，
然后在打包时再放入 slf4j-log4j12-version.jar 和 log4j.jar 就可以了。
 
现在还有一个问题，假如你正在开发应用程序所调用的组件当中已经使用了 JCL 的，还有一些组建可能直接调用了 java.util.logging，
这时你需要一个桥接器（名字为 XXX-over-slf4j.jar）把他们的日志输出重定向到 SLF4J，所谓的桥接器就是一个假的日志实现工具，
比如当你把 jcl-over-slf4j.jar 放到 CLASS_PATH 时，即使某个组件原本是通过 JCL 输出日志的，现在却会被 jcl-over-slf4j “骗到”SLF4J 里，
然后 SLF4J 又会根据绑定器把日志交给具体的日志实现工具。过程如下
 
---

Component
   |
   | log to Apache Commons Logging
   V
 jcl-over-slf4j.jar --- (redirect) ---> SLF4j ---> slf4j-log4j12-version.jar ---> log4j.jar ---> 输出日志
 
看到上面的流程图可能会发现一个有趣的问题，假如在 CLASS_PATH 里同时放置 log4j-over-slf4j.jar 和 slf4j-log4j12-version.jar 会发生什么情况呢？
没错，日志会被踢来踢去，最终进入死循环。
 
所以使用 SLF4J 的比较典型搭配就是把 slf4j-api、JCL 桥接器、java.util.logging（JUL）桥接器、log4j 绑定器、log4j 这5个 jar 放置在 CLASS_PATH 里。

--- 
### 使用案例

使用的jar包

commons-logging
案例如下：

private static Log logger=LogFactory.getLog(JulJclTest.class);

public static void main(String[] args){
    if(logger.isTraceEnabled()){
        logger.trace("commons-logging-jcl trace message");
    }
}
可以看到我们使用commons-logging的API来进行日志的编程操作，现在想切换成logback来进行日志的输出（这其实就是commons-logging与logback的集成）

解决办法如下：

第一步：去掉commons-logging jar包（其实去不去都无所谓）

第二步：加入以下jar包：

jcl-over-slf4j（实现commons-logging切换到slf4j）
slf4j-api
logback-core
logback-classic
第三步：在类路径下加入logback的配置文件

### 切换原理
这个原理之前都已经说过了，可以看下commons-logging与logback的集成

就是commons-logging通过jcl-over-slf4j 来选择slf4j作为底层的日志输出对象，而slf4j又选择logback来作为底层的日志输出对象。


## 冲突说明
仍然是这里的内容slf4j官网的冲突说明

其实明白上面介绍的各jar包的作用，就很容易理解

1. jcl-over-slf4j 与 slf4j-jcl 冲突
jcl-over-slf4j： commons-logging切换到slf4j

slf4j-jcl : slf4j切换到commons-logging

如果这两者共存的话，必然造成相互委托，造成内存溢出

2. log4j-over-slf4j 与 slf4j-log4j12 冲突
log4j-over-slf4j ： log4j1切换到slf4j
slf4j-log4j12 : slf4j切换到log4j1
如果这两者共存的话，必然造成相互委托，造成内存溢出。但是log4j-over-slf4内部做了一个判断，可以防止造成内存溢出：

即判断slf4j-log4j12 jar包中的org.slf4j.impl.Log4jLoggerFactory是否存在，如果存在则表示冲突了，抛出异常提示用户要去掉对应的jar 包，代码如下，在slf4j-log4j12 jar包的org.apache.log4j.Log4jLoggerFactory中：

Java日志框架slf4j、jcl、jul、log4j1、log4j2、logback大总结

3. jul-to-slf4j 与 slf4j-jdk14 冲突
jul-to-slf4j ： jdk-logging切换到slf4j
slf4j-jdk14 : slf4j切换到jdk-logging
如果这两者共存的话，必然造成相互委托，造成内存溢出

## 引用资料：
> https://blog.csdn.net/yycdaizi/article/details/8276265
