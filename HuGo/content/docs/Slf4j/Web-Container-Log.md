# Web 容器 日志桥接到 Slf4j

## 背景

Web 容器内的日志由容器自己管理，会输出到自己的日志文件中，应用会有自己的输出方式，当项目报错时，无法知道错误来源是导致还是应用导致，排查路径比较多。

如果能统一 容器和应用的日志输出，则可以保证在统一一个日志收集工具上进行查看，异常日志看起来会更加明了。比如：

- `Request header is too large`
- 等

## 概要

- 常见 Web 容器使用 JDK 内置的` jul` 日志组件 ` java.util.logging.Logger`
- 所有的 `Logger` 通过 ` java.util.logging.LogManager` 进行管理，包括 `rootLogger` （logger 名是空字符串）
- `LogManager` 在初始化的时候，通过 `static` 静态代码块，读取 `java.util.logging.manager` 系统属性 确定 `LogManager` 的实现类，如果没有指定，就把自己作为默认实现
  - **Tomcat** 
    - 在 `bin/catalina.sh` 中 通过 `LOGGING_MANAGER` 进行设置
    - 实现类是 `org.apache.juli.ClassLoaderLogManager`
  - **Resin**
    - `com.caucho.boot.ResinBoot` > `com.caucho.boot.WatchdogManager` ，通过 Java 读取配置，构造启动参数，最终唤醒 `com.caucho.server.resin.Resin` 进程
    - `-Djava.util.logging.manager=com.caucho.log.LogManagerImpl` 是写死在 ResinBoot 代码流程中的



## java.util.logging.LogManager

- JDK 内置的 `LogManager` 管理所有的 `Logger`
  - `LogManager.LoggerContext.Hashtable<String,LoggerWeakRef>`
- `Logger` 通过属性 `parent` 和 `useParentHandlers` 维护层级关系和决定是否使用 parent 的 `Handler` 输出日志
- `java.util.logging.Handler` 类似于 Appender ，是真正输出日志的地方，JDK 内置的 有 `java.util.logging.ConsoleHandler` 、`java.util.logging.FileHandler` 等
- `jul`（`java.util.logging` ）默认读取 `jre/lib/logging.properties` 文件中的配置

 

## org.slf4j:jul-to-slf4j

- `slf4j` 提供了 JDK Logging 桥接到 slf4j 的工具，不过需要手动调用，调用方式如下
  - 判断是否安装：`org.slf4j.bridge.SLF4JBridgeHandler.isInstalled()`
  - 卸载：`org.slf4j.bridge.SLF4JBridgeHandler.uninstall()`
  - 安装：`org.slf4j.bridge.SLF4JBridgeHandler.install()`
- 原理是在 `rootLogger`（`LogManager.getLogManager().getLogger("")`）中新增 `SLF4JBridgeHandler`，重写日志输出实现，输出时调用 `slf4j-api` 进行输出，之后便可通过 slf4j 适配各种日志框架。
- Spring Boot 项目在启动过程中已经自动把过程处理掉了，所以无需关注

 

## org.apache.juli.ClassLoaderLogManager

`juli`（可能是 **j**ava **u**til **l**ogg**i**ng 的缩写）为了进行不同 webapp 之间的隔离， `ClassLoaderLogManager extends LogManager`，通过 不同应用的 `ClassLoader` 对 `Logger` 进行隔离。

> - `org.apache.juli.ClassLoaderLogManager`
>   - `ClassLoaderLogManager` 属性： `Map<ClassLoader, ClassLoaderLogInfo> classLoaderLoggers`
>     - `ClassLoaderLogInfo` 属性：`Map<String, Logger> loggers`
>     - `ClassLoaderLogInfo` 属性：`Map<String, Handler> handlers`

 Tomcat 自身的 ClassLoader 是 `URLClassLoader`，但是用 webapp 的 ClassLoader 是 `ParallelWebappClassLoader`，从  `ClassLoaderLogManager` 中获取 Logger 的时候，因为当前线程的 ClassLoader 不一样，获取到的 RootLogger 其实也不是同一个对象。

但是 `ClassLoaderLogManager` 对象在当前 JVM 中只会又一个，如果可以把 `SLF4JBridgeHandler` 添加到 Tomcat 子什么的 rootLogger 中的，则 Tomcat 输出日志的时候，就可以把日志 输出到 slf4j。

以下示例代码供参考：

```java
private static void tomcatSlf4jBridgeHandlerInstall() throws IllegalAccessException, NoSuchFieldException {
    final LogManager logManager = LogManager.getLogManager();

    final String currentLogManager = LogManager.getLogManager().getClass().getName();
    final String tomcatLogManager = "org.apache.juli.ClassLoaderLogManager";

    // 判断当前 LogManager 是否是 Tomcat 默认的 LogManager
    if (!Objects.equals(currentLogManager, tomcatLogManager)) {
        return;
    }

    // 获取 Map<ClassLoader, ClassLoaderLogInfo> classLoaderLoggers
    final Field classLoaderLoggersField = logManager.getClass().getDeclaredField("classLoaderLoggers");
    classLoaderLoggersField.setAccessible(true);
    final Map classLoaderLoggers = (Map) classLoaderLoggersField.get(logManager);

    // 遍历 Map<ClassLoader, ClassLoaderLogInfo> classLoaderLoggers
    final Set entrySet = classLoaderLoggers.entrySet();
    for (Object entry : entrySet) {

        // ClassLoader <-> ClassLoaderLogInfo
        @SuppressWarnings("All") final Map.Entry<ClassLoader, Object> classLoaderAndLogInfo = (Map.Entry) entry;

        // 判断是否是 Tomcat 的 ClassLoader
        if (!Objects.equals(classLoaderAndLogInfo.getKey().getClass(), URLClassLoader.class)) {
            continue;
        }

        final Object logInfo = classLoaderAndLogInfo.getValue();
        final Field loggersField = logInfo.getClass().getDeclaredField("loggers");
        loggersField.setAccessible(true);

        // Map<String, Logger> loggers
        final Map o = (Map) loggersField.get(logInfo);
        final Logger rootLogger = (Logger) o.get("");

        // 使用 org.slf4j.bridge.SLF4JBridgeHandler
        java.util.logging.Handler[] handlers = rootLogger.getHandlers();
        if (Objects.isNull(handlers)) {
            return;
        }
        // 判断是否已经安装过
        for (Handler handler : handlers) {
            if (Objects.equals(handler.getClass(), SLF4JBridgeHandler.class)) {
                return;
            }
        }
        // 安装
        rootLogger.addHandler(new SLF4JBridgeHandler());

        // org.apache.juli.logging.Log.UserDataHelper 非法信息通过 INFO 级别打印的几种方式（默认是 INFO_THEN_DEBUG 级别）
        // System.setProperty("org.apache.juli.logging.UserDataHelper.CONFIG", "INFO_ALL");
        // 修改抑制时间，默认是 24小时，即非法信息，首次是 INFO 级别，之后24小时内都会变成 DEBUG 级别
        // System.setProperty("org.apache.juli.logging.UserDataHelper.SUPPRESSION_TIME", "0");

        /*
         * 完全接管指定的 Logger
         */
        List<String> loggerNames = Arrays.asList("org.apache.coyote.http11.Http11Processor");
        final String loggerNamesConfig = System.getProperty("tomcat.logger.handler.override.names");
        if (Objects.nonNull(loggerNamesConfig)) {
            loggerNames = Arrays.asList(loggerNamesConfig.split(","));
        }

        for (String loggerName : loggerNames) {
            // 设置原始 Logger 日志级别，具体是否输出由 Slf4j 控制
            final Logger logger = (Logger) o.get(loggerName);
            if (null == logger) {
                continue;
            }
            logger.setLevel(Level.ALL);
            for (Handler handler : handlers) {
                logger.removeHandler(handler);
            }
            logger.addHandler(new SLF4JBridgeHandler());
            logger.setUseParentHandlers(false);
        }

    }
}
```



## com.caucho.log.LogManagerImpl

Resin 项目隔离相对 Tomact 要更复杂一些，Logger 的获取基本沿用 JDK 的方式，没有使用 ClassLoader 进行区分，但是 Handler 的寻址基本上重写了 JDK 的默认实现，这个可能跟整个 Resin 的隔离机制有关，大部分的变量都是通过 `EnvironmentLocal` 重新进行封装的的， `EnvironmentLocal` 在初始化的时候会通过全局自增 ID 分配一个唯一的 `varName` 表示一个唯一的变量，变量会放在 `EnvironmentClassLoader` 的属性中，通过重写变量的获取方式来达到隔离的目的。

> Resin 的日志级别设计 感觉 比 Tomcat 也更合理一些（某些场景下），比如 `Request header is too large` 错误，在 Tomcat 下，第一次是 INFO 界别，之后再 24 小时的抑制时间内，会变成 DEBUG 级别，可能不会输出任何错误日志；而 Resin 始终都是 WARN 级别，来告知有异常的请求

以下示例代码供参考：

```java
private static void resinSlf4jBridgeHandlerInstall() throws ReflectiveOperationException {
    // com.caucho.log.EnvironmentLogger
    final Logger rootLogger = LogManager.getLogManager().getLogger("");
    final String resinLoggerName = "com.caucho.log.EnvironmentLogger";
    if (!resinLoggerName.equals(rootLogger.getClass().getName())) {
        return;
    }

    final Field localHandlersFiled = rootLogger.getClass().getDeclaredField("_localHandlers");
    localHandlersFiled.setAccessible(true);
    // EnvironmentLocal<Handler[]> _localHandlers
    final Object localHandlersEnv = localHandlersFiled.get(rootLogger);
    if (Objects.isNull(localHandlersEnv)) {
        return;
    }

    /*
     * 方式1：获取 localHandlersEnv 中 _varName，设置到 Handler 到 EnvironmentClassLoader[resin-system:] 的 _attributes 中
     * 原理详见：EnvironmentLogger.getHandlers() 和 EnvironmentLocal.get() 方法
     */
    final Field varNameField = localHandlersEnv.getClass().getDeclaredField("_varName");
    varNameField.setAccessible(true);
    final String handlerVarName = (String) varNameField.get(localHandlersEnv);

    final String environmentClName = "com.caucho.loader.EnvironmentClassLoader";
    final String rootDynamicCltName = "com.caucho.loader.RootDynamicClassLoader";
    ClassLoader currentCl = Thread.currentThread().getContextClassLoader();
    ClassLoader parentCl = currentCl.getParent();
    for (; Objects.nonNull(parentCl); currentCl = parentCl, parentCl = currentCl.getParent()) {
        final boolean currentIsEnv = Objects.equals(currentCl.getClass().getName(), environmentClName);
        final boolean parentIsRootDynamic = Objects.equals(parentCl.getClass().getName(), rootDynamicCltName);
        if (currentIsEnv && parentIsRootDynamic) {
            final Method setAttributeMethod = currentCl.getClass().getDeclaredMethod("setAttribute", String.class, Object.class);
            setAttributeMethod.setAccessible(true);
            setAttributeMethod.invoke(currentCl, handlerVarName, Collections.singleton(new SLF4JBridgeHandler()).toArray(new Handler[0]));
            return;
        }
    }

    /*
     * 方式2： 修改 rootLogger Handler 的  _globalValue 值
     */
    // E _globalValue
    final Field globalValueField = localHandlersEnv.getClass().getDeclaredField("_globalValue");
    globalValueField.setAccessible(true);
    final Handler[] globalHandlers = (Handler[]) globalValueField.get(localHandlersEnv);
    if (Objects.nonNull(globalHandlers)) {
        final ArrayList<Handler> globalHandlerList = new ArrayList<>(Arrays.asList(globalHandlers));
        // 新增 SLF4JBridgeHandler
        globalHandlerList.add(new SLF4JBridgeHandler());
        // 放回老位置
        globalValueField.set(localHandlersEnv, globalHandlerList.toArray(new Handler[0]));
    }
}
```



## 换个角度思考

从上面分析可以看出，不管是修改 Tomcat 还是 Resin 的默认日志行为，侵入性都比较大（可能是我没找到正确的方式），需要用到 大量的反射，而且还突破了 容器的 隔离限制，风险其实蛮大的，可能会造成日志大量的输出或者不输出。

我们再回顾最初的问题，对于非法请求， 其实 Web 容器会返回 **错误页面** 和 **状态码** 给调用方，调用方报错时，应该打印出原因，而不是吃掉日志，或者打印一些不便于排查的信息，或者依赖服务方的日志，实际上是不合理的。

 

 

 