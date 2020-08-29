### Log4j2 结构

log4j2是apache在log4j的基础上，参考logback架构实现的一套新的日志系统（我感觉是apache害怕logback了）。
log4j2的[官方文档](https://logging.apache.org/log4j/2.0/index.html)上写着一些它的优点：

- 在拥有全部logback特性的情况下，还修复了一些隐藏问题
- API 分离：现在log4j2也是门面模式使用日志，默认的日志实现是log4j2，当然你也可以用logback（应该没有人会这么做）
- 性能提升：log4j2包含下一代基于LMAX Disruptor library的异步logger，在多线程场景下，拥有18倍于log4j和logback的性能
- 多API支持：log4j2提供Log4j 1.2, SLF4J, Commons Logging and java.util.logging (JUL) 的API支持
- 避免锁定：使用Log4j2 API的应用程序始终可以选择使用任何符合SLF4J的库作为log4j-to-slf4j适配器的记录器实现
- 自动重新加载配置：与Logback一样，Log4j 2可以在修改时自动重新加载其配置。与Logback不同，它会在重新配置发生时不会丢失日志事件。
- 高级过滤： 与Logback一样，Log4j 2支持基于Log事件中的上下文数据，标记，正则表达式和其他组件进行过滤。
- 插件架构： Log4j使用插件模式配置组件。因此，您无需编写代码来创建和配置Appender，Layout，Pattern Converter等。Log4j自动识别插件并在配置引用它们时使用它们。
- 属性支持：您可以在配置中引用属性，Log4j将直接替换它们，或者Log4j将它们传递给将动态解析它们的底层组件。
- Java 8 Lambda支持
- 自定义日志级别
- 产生垃圾少：在稳态日志记录期间，Log4j 2 在独立应用程序中是无垃圾的，在Web应用程序中是低垃圾。这减少了垃圾收集器的压力，并且可以提供更好的响应时间性能。
- 和应用server集成：版本2.10.0引入了一个模块log4j-appserver，以改进与Apache Tomcat和Eclipse Jetty的集成。



![image.png](https://ata2-img.cn-hangzhou.oss-pub.aliyun-inc.com/df119ab6cb37026248f9449ea395e73c.png)





 - LogManager

   > 根据配置指定LogContexFactory，初始化对应的LoggerContext

  

​		