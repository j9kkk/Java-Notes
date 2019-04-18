# Java中的日志框架

## 常见日志框架

`JUL`|`Log4j1`|`Log4j2`|`Logback`|`JCL`|`Slf4j`

## 日志框架需要解决的问题

* 稳定性高：不可影响主进程的正常运行且自身的日志内容准确无歧义
* 扩展性高：对于不同的日志输出需求，有便捷的方式进行自定义扩展
* 开销低：不可占用服务器的过多资源，影响主进程的执行速度
* 延迟低：对应的日志内容输出不可延迟太久，否则失去观察的意义

## 框架对比

| 框架    | 版本              | 优点                                                        | 缺点                               | 作者                                                         | 备注                                                         |
| ------- | ----------------- | ----------------------------------------------------------- | ---------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| JUL     | @since JDK 1.4    | JDK自带日志工具类，无需额外依赖                             | 功能单一                           | [Oracle](https://docs.oracle.com/javase/8/docs/api/index.html) | java.util.logging；提供了基础的Handler(Appender)，Formatter(Pattern)等组件化功能 |
| Logback | 2006-2018         | 优化了Log4j1中的缺陷及不足；配合Slf4j使用时不需要引入适配层 | 重载配置文件时可能丢失日志         | [QOS.ch](https://logback.qos.ch/)                            | 出现时间介于Log4j1与Log4j2，是基于Slf4j标准的***原生实现***，相比其他框架不需要引入适配层 |
| Log4j1  | 1999-2012         | 标准的日志接口；模块化的设计理念；                          | 多线程下可能存在死锁               | [Apache](http://logging.apache.org/log4j/1.2/)               | [2015年被Apache声明不再维护](https://blogs.apache.org/foundation/entry/apache_logging_services_project_announces)，最后版本为2012年发布的log4j 1.2.17 |
| Log4j2  | 2012-2019         | 更少的内存占用；更高的并发性能；更完善的使用手册            | 特性繁多，完全掌握需要一定学习成本 | [Apache](https://logging.apache.org/log4j/2.x/)              | 在1的版本上完全重写，基于[LMAX Disruptor](https://www.jianshu.com/p/a44b779c22cb)库使得并发性能大幅提升 |
| JCL     | 2005-2014         | Apache 官方项目                                             | 使用不当易存在内存泄漏             | [Apache](http://commons.apache.org/proper/commons-logging/guide.html) | Apache Commons Logging；日志抽象接口层，最新版截止2014年；因设计理念及使用方式导致在某些情况下存在内存泄漏的问题 |
| Slf4j   | 2009-2019 >=1.6.0 | 易用，单jar包，使用范围广                                   |                                    | [QOS.ch](https://www.slf4j.org/)                             | Simple Logging Facade for Java日志抽象接口层                 |

## Slf4j

- 保证了项目内日志框架升级的便捷性，项目间日志框架的一致性
- 利用`Bridging legacy logging APIs`实现已有JCL、JUL、Log4j多项目的归并统一
- 参数化日志打印
- 无绑定、多绑定、版本异常等可以在加载期进行检测提示

### 解决的问题

![](https://ws4.sinaimg.cn/large/006tNc79gy1g26ljdiexnj312q0o2n29.jpg)

![](https://ws1.sinaimg.cn/large/006tNc79gy1g26ljwxrm9j31300okgqw.jpg)

### 调用关系链

![](https://www.slf4j.org/images/concrete-bindings.png)

## Log4j2

### 性能对比

![](https://logging.apache.org/log4j/2.x/images/async-vs-sync-throughput.png)

![](https://logging.apache.org/log4j/2.x/images/async-throughput-comparison.png)

### 基础概念

#### Log Level

`OFF`|`FATAL`|`ERROR`|`WARN`|`INFO`|`DEBUG`|`TRACE`|`ALL`

#### Log Event

| Event Level | LoggerConfig Level |       |      |      |       |       |      |
| ----------- | :----------------: | ----- | ---- | ---- | ----- | ----- | ---- |
|             |       TRACE        | DEBUG | INFO | WARN | ERROR | FATAL | OFF  |
| ALL         |        YES         | YES   | YES  | YES  | YES   | YES   | NO   |
| TRACE       |        YES         | NO    | NO   | NO   | NO    | NO    | NO   |
| DEBUG       |        YES         | YES   | NO   | NO   | NO    | NO    | NO   |
| INFO        |        YES         | YES   | YES  | NO   | NO    | NO    | NO   |
| WARN        |        YES         | YES   | YES  | YES  | NO    | NO    | NO   |
| ERROR       |        YES         | YES   | YES  | YES  | YES   | NO    | NO   |
| FATAL       |        YES         | YES   | YES  | YES  | YES   | YES   | NO   |
| OFF         |         NO         | NO    | NO   | NO   | NO    | NO    | NO   |

#### Appender

真正执行日志输出的类，log4j2预定义了多种用途的Appender如Console Appender，File Appender，Http Appender等，其中Appender按执行层级又可以分为二种：普通Appender与引用Appender，引用Appender即自身并不实现具体的输出而是对普通Appender进行了一层包装来实现异步、过滤、转发等目的

#### Logger

具体的日志对象，一个Logger对象可以包含[0, n)个Appender来同时输出到不同流；同时Logger对象还包含一些管理信息如Log Level及Log Filter等

### 结构图解

![屏幕快照 2019-04-18 10.54.10](https://ws4.sinaimg.cn/large/006tNc79gy1g26lm0c33pj30u00zqai8.jpg)

### 常用配置项

```
<?xml version="1.0" encoding="UTF-8"?>;
<Configuration name="my configuration file" status="WARN" monitorInterval="30" desc="log4jdebug.log">
  <Properties>
    <Property name="name1">value</property>
    <Property name="name2" value="value2"/>
  </Properties>
  <filters>
      <MarkerFilter marker="FLOW" onMatch="ACCEPT" onMismatch="NEUTRAL"/>
      <MarkerFilter marker="EXCEPTION" onMatch="ACCEPT" onMismatch="DENY"/>
  </filters>
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/$${date:yyyy-MM}/app-%d{MM-dd-yyyy}-%i.log.gz">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
      <Policies>
        <TimeBasedTriggeringPolicy />
        <SizeBasedTriggeringPolicy size="250 MB"/>
      </Policies>
    </RollingFile>
    ...
  </Appenders>
  <Loggers>
    <Logger name="name1">
      <filter  ... />
      <AppenderRef ref="name1"/>
      <AppenderRef ref="name2"/>
    </Logger>
    ...
    <Root level="level">
      <AppenderRef ref="name"/>
    </Root>
  </Loggers>
</Configuration>
```

#### Configuration

* status：LogLevel；log4j2内部代码日志级别，调试时可设置为trace
* monitorInterval：配置变更检测间隔，单位秒；注意只有当monitorInterval之后并有新的logEvent时才会真正触发reconfiguration
* dest：err|out|file|URL；log4j2内部代码输出流，对于不方便直接查看的环境可以导出调试信息至文件

#### Properties

类似POM文件中的Properties，定义通过kvp，引用通过${name}；除此之外在引用时还可以通过特定的前缀来指定引用的值的格式`${prefix:name}`，如`${base64:SGVsbG8gV29ybGQhCg==}`等价于`Hello World!`，`${sys:some.property:-default_value}`表示取名为`some.property`的系统参数，当不存在时使用`default_value`进行替换

#### Filter

Log Event过滤器，每个过滤器有三种返回结果`Accept`|`Deny`|`Neutral`，分别表示直接接受，直接拒绝，向下传递；根据Filter的作用域又可以分为以下三种

* 全局Filter，配置节点与Properties\Appenders\Loggers同级
* Logger Filter，位于Logger中，针对某个具体的Logger进行过滤
* Appender Filter，位于Appender中，针对某个具体的Appender进行过滤

#### Appenders

##### Rolling File Appender

| Parameter Name                                             | Type               | Values       | Default | Description                                                  |
| :--------------------------------------------------------- | :----------------- | ------------ | ------- | :----------------------------------------------------------- |
| append                                                     | boolean            | true\|false  | true    | 新日志附加至文件末尾或全量覆盖                               |
| bufferedIO                                                 | boolean            | true\|false  | true    | 是否开启文件写入缓存                                         |
| bufferSize                                                 | int                |              | 8192    | 配合bufferedIO使用，单位字节                                 |
| createOnDemand                                             | boolean            | true\|false  | false   | 是否开启延迟创建文件                                         |
| filter                                                     | Filter             |              |         | 过滤器，多个filter应使用filters标签                          |
| fileName                                                   | String             |              |         | 日志路径，若不存在则自动创建                                 |
| filePattern                                                | String             |              |         | 回滚的日志文件名格式                                         |
| immediateFlush                                             | boolean            | true\|false  | true    | 是否立即写入磁盘                                             |
| layout                                                     | Layout             |              | %m%n    | 日志内容格式，参考[Pattern Layout](https://logging.apache.org/log4j/2.x/manual/layouts.html#Pattern_Layout) |
| name                                                       | String             |              |         | 同配置集中，Appender的name必须唯一                           |
| policy                                                     | TriggeringPolicy   |              |         | 滚动触发策略，决定何时进行文件滚动                           |
| policy.OnStartupTriggeringPolicy.minSize                   | long               |              | 1       | 滚动文件大小最小值                                           |
| policy.SizeBasedTriggeringPolicy.size                      | String             | 20KB\|MB\|GB |         | filePattern中必须包含%i项，否则会导致文件直接被覆盖          |
| policy.TimeBasedTriggeringPolicy.interval                  | int                | 1            |         | 基于filePattern中的日期精度单位触发滚动                      |
| policy.TimeBasedTriggeringPolicy.modulate                  | boolean            | true\|false  |         | 是否使用绝对时间                                             |
| policy.TimeBasedTriggeringPolicy.maxRandomDelay            | int                |              | 0       | 触发滚动时随机延迟N秒，避免多触发下造成CPU波峰               |
| policy.CronTriggeringPolicy.schedule                       | String             |              |         | cron表达式                                                   |
| policy.CronTriggeringPolicy.evaluateOnStartup              | boolean            |              |         | 是否启动时候立即执行                                         |
| strategy                                                   | RolloverStrategy   |              |         | 滚动执行策略，决定怎么进行文件滚动                           |
| strategy.DefaultRolloverStrategy.fileIndex                 | String             | min\|max     | max     | 备份的文件按时间降序或升序排号，默认升序即编号最大的时间最近 |
| strategy.DefaultRolloverStrategy.min                       | int                |              | 1       | 备份文件排号起点                                             |
| strategy.DefaultRolloverStrategy.max                       | int                |              | 7       | 备份文件排号最大值，超出最大值时候将删除时间最远的文件       |
| strategy.DefaultRolloverStrategy.compressionLevel          | int                | [0-9]        | 0       | 只有当filePattern配置后缀为压缩时生效，0-不压缩，1-9表示压缩率 |
| strategy.DefaultRolloverStrategy.tempCompressedFilePattern | String             |              |         | 压缩期间使用的临时文件名                                     |
| strategy.DefaultRolloverStrategy.delete                    | Delete             |              |         | 执行滚动时自定义的删除行为                                   |
| strategy.DefaultRolloverStrategy.posixViewAttribute        | posixViewAttribute |              |         | 执行滚动时自定义的文件权限                                   |
| ignoreExceptions                                           | boolean            | true\|false  | true    | 是否忽略appender的内部异常                                   |
| filePermissions                                            | String             |              |         | 创建文件时赋予的权限，POSIX格式                              |
| fileOwner                                                  | String             |              |         | 文件所属人                                                   |
| fileGroup                                                  | String             |              |         | 文件所属组                                                   |

----

> 触发策略Policy配置时类似Filter，可以使用Policies进行多项配置，只要任一项Policy满足条件则触发

```
<Policies>
  <OnStartupTriggeringPolicy />
  <SizeBasedTriggeringPolicy size="20 MB" />
  <TimeBasedTriggeringPolicy />
</Policies>
```

##### AsyncAppender

```
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <File name="MyFile" fileName="logs/app.log">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
    </File>
    <Async name="Async">
      <AppenderRef ref="MyFile"/>
    </Async>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="Async"/>
    </Root>
  </Loggers>
</Configuration>
```

| Parameter Name       | Type                 | Default            | Description                                                  |
| :------------------- | :------------------- | ------------------ | :----------------------------------------------------------- |
| AppenderRef          | String               |                    | 关联的appender.name                                          |
| blocking             | boolean              | true               | 内存队列满时是等待还是写入errorRef                           |
| shutdownTimeout      | integer              | 0                  |                                                              |
| bufferSize           | integer              | 1024               | 缓冲大小，单位字节；                                         |
| errorRef             | String               |                    | 执行异常时候输出的appender.name                              |
| filter               | Filter               |                    | 同样可以使用filters进行多项组合                              |
| name                 | String               |                    | 唯一标识                                                     |
| ignoreExceptions     | boolean              | true               | 是否忽略内部异常                                             |
| includeLocation      | boolean              | false              | 是否记录caller location即调用堆栈                            |
| BlockingQueueFactory | BlockingQueueFactory | ArrayBlockingQueue | This element overrides what type of `BlockingQueue` to use. See[below documentation](https://logging.apache.org/log4j/2.x/manual/appenders.html#BlockingQueueFactory) for more details. |

##### RewriteAppender

> 重写log event，主要用于数据过滤或脱敏

##### RoutingAppender

> appender重定向，需要注意的是routing必须定义在所有关联appender之后

#### Logger

* additivity：true|false；是否继承父类logger，默认继承
* name：string；唯一标识，除root logger外都必须配置
* level：log level；日志输出级别，默认为error
* appenderRef：string；关联appender的name

### 合并配置项

* 使用XInclude`<xi:include href="log4j-xinclude-appenders.xml" />`进行文件内合并
* 使用log4j.configurationFile参数进行跨文件合并：file1,file2

### 注意事项

* 当未提供log4j.configurationFile启动参数时，将按内置优先级依次查找配置文件，都未找到的情况下使用默认ConsoleAppender且Level设置为Error
* 调试log4j2的内部日志有二种常用方式：设置配置文件的status属性为trace；在启动参数中加入log4j2.debug（仅支持debug级别）；更多log4j2支持的启动参数请查阅[这里](https://logging.apache.org/log4j/2.x/manual/configuration.html#System_Properties)

## FAQ

> 为什么我使用了日志配置文件确依然没有日志输出？

答：

* 确认是否引入了slf4j的实现包，比如slf4j-log4j-impl；若没有，slf4j会提示无法找到对应实现类，若提供了多个slf4j实现包，则同样会提示绑定冲突
* 确认是否正确提供了日志配置文件；若没有，log4j会提示找不到配置文件并启动默认配置集（Console + Level.Error）
* 确认是否配置了bufferIO及缓冲区；只有缓冲区满才会提交到磁盘IO进行写入操作
* 确认是否有磁盘文件创建权限；可以使用sudo启动或预先以运行用户的角色建立好日志文件路径

> 我添加了依赖slf4j-log4j2-impl，那么我还是否需要额外引入slf4j-api？

答：不需要，slf4j的实现包具体依赖项以POM文件为准

----

> 分析如下三种函数使用方式，哪种最优，好在哪里？

```
logger.debug("Entry number: " + i + " is " + String.valueOf(entry[i]));
```

```
if(logger.isDebugEnabled()) {
  logger.debug("Entry number: " + i + " is " + String.valueOf(entry[i]));
}
```

```
logger.debug("Entry number: {} is {}.", i, String.valueOf(entry[i]);
```

答：第三种，减少字符串的合并操作

----

> 在log4j的参数化日志方法中，分析如下几种情况的输出

```
logger.debug("param-1: {}, param-2: {}, param-3: {}", "1", "2", "3")
```

```
logger.info("param-1: \\{}, param-2: {}, param-3: {}", "1", "2", "3")
```

```
logger.info("param-1: {}, param-2: {{}}, param-3: {}", "1", "2", "3")
```

答：

* param-1: 1, param-2: 2, param-3: 3
* param-1: {}, param-2: 1, param-3: 2
* param-1: 1, param-2: {2}, param-3: 3

----

> Logger对象定义为static或variable有什么区别，适用于哪些场景？JCL及Slf4j是如何解决这个问题的？

![屏幕快照 2019-04-18 10.56.10](https://ws1.sinaimg.cn/large/006tNc79gy1g26lo7hu23j30xy0jkdk1.jpg)

答：static在同容器多应用的场景下可能存在引用冲突；JCL默认使用MAP来存储每个Logger的引用，需要手动释放可能存在使用不当导致内存泄漏；Slf4j没有这样的机制，是否static完全交由使用者控制

----

> Slf4j为什么没有FATAL以及TRACE级别？

答：Slf4j的作者设计理念，认为FATAL类似ERROR，TRACE类似DEBUG，存在概念上的混淆；如果确实需要标记为FATAL或TRACE可以使用Marker + Pattern来实现

----

> 分析以下配置文件最终生成的日志文件将会是什么样的？

```
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" filePattern="logs/app-%d{yyyy-MM-dd-HH}-%i.log.gz">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
      <Policies>
        <CronTriggeringPolicy schedule="0 0 * * * ?"/>
        <SizeBasedTriggeringPolicy size="250 MB"/>
      </Policies>
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

答：

## 资源链接

[JUL - java.util.logging - ORACLE](https://docs.oracle.com/en/java/javase/11/docs/api/java.logging/java/util/logging/package-summary.html)

[JUL Tutorials - vogella.com](https://www.vogella.com/tutorials/Logging/article.html)

[StackOverflow - Why NOT JUL?](https://stackoverflow.com/questions/11359187/why-not-use-java-util-logging)

[JCL - Apache Commons Logging](http://commons.apache.org/proper/commons-logging/)

[Slf4j - Simple Logging Facade for Java](<https://www.slf4j.org/>)

[Slf4j - 如何正确的排除依赖包中的日志框架](https://www.slf4j.org/faq.html#excludingJCL)

[Slf4j - 如何提高日志的性能](https://www.slf4j.org/faq.html#logging_performance)

[Logger - Static or Not?](https://wiki.apache.org/commons/Logging/StaticLog)

[Log4j 1.x to Log4j 2.x](https://logging.apache.org/log4j/2.x/manual/migration.html)

[Log4j 2 Benchmarks](https://logging.apache.org/log4j/2.x/performance.html#benchmarks)

[Log4j 2 Pattern Layout](https://logging.apache.org/log4j/2.x/manual/layouts.html#Pattern_Layout)