> 本文档主要介绍 sprintboot 如何通过 `SpringApplication.run(MainClass.class, args);` 完成的项目启动的过程
>
> 使用的是SpringBoot版本号: 2.7.11

主要执行的启动函数

```java
/**
 * Static helper that can be used to run a {@link SpringApplication} from the
 * specified sources using default settings and user supplied arguments.
 * @param primarySources the primary sources to load
 * @param args the application arguments (usually passed from a Java main method)
 * @return the running {@link ApplicationContext}
 */
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
    return new SpringApplication(primarySources).run(args);
}
```

### SpringApplication对象

> new SpringApplication(primarySources)

构造函数如下, 入参如下说明:

- resourceLoader: 传入为 null, 后续启动过程中会通过方法 `getDefaultClassLoader` 获取启动过程使用的默认类加载器对象;
- primarySources: SpringBoot启动使用的主类, 可以近似认为是: `MainClass.class`;

```java
/**
 * Create a new {@link SpringApplication} instance. The application context will load
 * beans from the specified primary sources (see {@link SpringApplication class-level}
 * documentation for details). The instance can be customized before calling
 * {@link #run(String...)}.
 * @param resourceLoader the resource loader to use
 * @param primarySources the primary bean sources
 * @see #run(Class, String[])
 * @see #setSources(Set)
 */
@SuppressWarnings({ "unchecked", "rawtypes" })
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    this.bootstrapRegistryInitializers = new ArrayList<>(
            getSpringFactoriesInstances(BootstrapRegistryInitializer.class));
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

#### 1. 基础信息保存

```java
// 保存传入的类加载器对象
this.resourceLoader = resourceLoader;
// 断言: 主启动类不能为空, 非空判断
Assert.notNull(primarySources, "PrimarySources must not be null");
// 将主启动类以链表的形式进行保存
this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
```

#### 2. web类型推断

>  使用方法: this.webApplicationType = WebApplicationType.deduceFromClasspath();

类型方法值返回:

- `None`:
  - The application should not run as a web application and should not start an embedded web server
  - 非 web 类型;
- `SERVLET`
  - The application should run as a servlet-based web application and should start an embedded servlet web server
  - Servlet web 类型, 并以嵌入式的形式进行启动;
- `REACTIVE`
  - The application should run as a reactive web application and should start an embedded reactive web server.
  - Reactive web 类型, 并以嵌入式的形式进行启动;

代码实现如下:

- 通过 `SERVLET_INDICATOR_CLASSES` `WEBFLUX_INDICATOR_CLASS` `WEBMVC_INDICATOR_CLASS` `JERSEY_INDICATOR_CLASS` 是否在classpath中是否存在，完成对类型的判断;

```java
private static final String[] SERVLET_INDICATOR_CLASSES = { "javax.servlet.Servlet",
"org.springframework.web.context.ConfigurableWebApplicationContext" };

private static final String WEBMVC_INDICATOR_CLASS = "org.springframework.web.servlet.DispatcherServlet";

private static final String WEBFLUX_INDICATOR_CLASS = "org.springframework.web.reactive.DispatcherHandler";

private static final String JERSEY_INDICATOR_CLASS = "org.glassfish.jersey.servlet.ServletContainer";

static WebApplicationType deduceFromClasspath() {
    if ( ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) && 
        !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null) && 
        !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
        return WebApplicationType.REACTIVE;
    }
    for (String className : SERVLET_INDICATOR_CLASSES) {
        if (!ClassUtils.isPresent(className, null)) {
            return WebApplicationType.NONE;
        }
    }
    return WebApplicationType.SERVLET;
}
```



#### 3. 容器注册

#### 4. 主启动类推断

> 使用方法: `this.mainApplicationClass = deduceMainApplicationClass();`

- 通过 Runtime 栈获取正在运行的方法集合;
- 获取到 `main` 方法, 并将主方法所在的类获取

```java
private Class<?> deduceMainApplicationClass() {
    try {
        StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
        for (StackTraceElement stackTraceElement : stackTrace) {
            if ("main".equals(stackTraceElement.getMethodName())) {
                return Class.forName(stackTraceElement.getClassName());
            }
        }
    }
    catch (ClassNotFoundException ex) {
        // Swallow and continue
    }
    return null;
}
```





### run方法

>  run(args)

zhangfan





### 附录

#### 1. 运行时栈信息

> StackTraceElement[] stackTrace = new RuntimeException().getStackTrace()

- 方法作用: 在方法执行时获取运行栈上的所有方法名称

- 测试方法:

```java
package com.zhangxiaofanfan;

public class Test {
    public static void main(String[] args) {
        method01();
    }

    public static void methodStack() {
       	StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
        // 在此处断点, 查看运行时栈 拍照
       	for (StackTraceElement stackTraceElement : stackTrace) {
            log.info(String.format("getMethodName is %s", stackTraceElement.getMethodName()));
       	}
    }

    public static void methodIn() { method01(); }

    public static void method01() { method02(); }

    public static void method02() { method03(); }

    public static void method03() { method04(); }

    public static void method04() { methodStack(); }
}
```

- 断点测试镜像

![get-cur-runtime-stack](https://raw.githubusercontent.com/zhangxiaofanfan/pic-floder/master/get-cur-runtime-stack.png)

- 测试结果:

```bash
15:27:57.094 [main] INFO com.zhangxiaofanfan.Test - getMethodName is methodStack
15:27:57.110 [main] INFO com.zhangxiaofanfan.Test - getMethodName is method04
15:27:57.110 [main] INFO com.zhangxiaofanfan.Test - getMethodName is method03
15:27:57.110 [main] INFO com.zhangxiaofanfan.Test - getMethodName is method02
15:27:57.110 [main] INFO com.zhangxiaofanfan.Test - getMethodName is method01
15:27:57.111 [main] INFO com.zhangxiaofanfan.Test - getMethodName is main
```

