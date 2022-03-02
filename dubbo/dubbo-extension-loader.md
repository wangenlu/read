# ExtensionLoader

`ExtensionLoader` 是 `Dubbo` [SPI](dubbo-spi.md) 的具体实现类，这里主要包含下面几个部分的分析：

1. 分析`ExtensionLoader`中的一些方法实现逻辑
2. 通过`ServiceConfig`中的`protocol.export`方法，分析 `dubbo` 自适应的实现方式

- [ExtensionLoader](#extensionloader)
  - [ExtensionLoader 的作用](#extensionloader-的作用)
  - [ExtensionLoader methods](#extensionloader-methods)
    - [getExtensionLoader](#getextensionloader)
    - [getExtension](#getextension)
    - [createExtension](#createextension)
    - [WrapperClass](#wrapperclass)
    - [injectExtension](#injectextension)
    - [loadExtensionClasses](#loadextensionclasses)
    - [getAdaptiveExtension](#getadaptiveextension)
    - [createAdaptiveExtensionClassCode](#createadaptiveextensionclasscode)
  - [Protocol](#protocol)
  - [Protocol\$Adaptive](#protocoladaptive)
  - [Summary](#summary)

## ExtensionLoader 的作用

- 进行扩展类的加载
- 进行扩展类的依赖注入
- 进行扩展类的包装
- 一些默认实现的自适应实现，比如 `Protocol`

## ExtensionLoader methods

下面通过`ExtensionLoader`中的方法来分析 `ExtensionLoader` 的实现

### getExtensionLoader

```java
// getExtensionLoader 是一个静态方法，参数是 type（比如：Protocol）
// 每一个类，都对应有自己的一个 ExtensionLoader 扩展点
// 还有自己的成员变量，比如 cachedWrapperClasses，这会导致生成的 Protocol 实例会进行包装，提供额外的扩展功能
// 后面会再次提到包装类
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    if (type == null) {
        throw new IllegalArgumentException("Extension type == null");
    }
    if (!type.isInterface()) {
        throw new IllegalArgumentException("Extension type(" + type + ") is not interface!");
    }
    // 检查是否有 SPI 这个注解
    if (!withExtensionAnnotation(type)) {
        throw new IllegalArgumentException("Extension type(" + type +
                ") is not extension, because WITHOUT @" + SPI.class.getSimpleName() + " Annotation!");
    }
    // 从 EXTENSION_LOADERS（ConcurrentMap）中获取，这个类对应的 ExtensionLoader
    // 如果存在就返回,不存在就new 一个
    // getExtensionLoader 是一个静态方法，同时共享了 EXTENSION_LOADERS 静态变量
    // 因此使用并发的 ConcurrentMap 的 putIfAbsent 方法来避免并发问题

    // 一种扩展点对应一个新的 ExtensionLoader 实例
    // 因此每一个扩展点的 cachedAdaptiveClass，cachedWrapperClasses 都是不一样的
    ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    if (loader == null) {
        EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
        loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    }
    return loader;
}
```

### getExtension

```java
// 这个方法根据 name 从缓存中 cachedInstances 拿对应的扩展点
// 如果为空，就创建一个，具体的实现在 createExtension 方法中
@SuppressWarnings("unchecked")
public T getExtension(String name) {
    if (name == null || name.length() == 0) {
        throw new IllegalArgumentException("Extension name == null");
    }
    if ("true".equals(name)) {
        return getDefaultExtension();
    }
    Holder<Object> holder = cachedInstances.get(name);
    if (holder == null) {
        cachedInstances.putIfAbsent(name, new Holder<Object>());
        holder = cachedInstances.get(name);
    }
    Object instance = holder.get();
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                instance = createExtension(name);
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}
```

### createExtension

```java
// getExtensionClasses 会从 cachedClasses 缓存中那对象
// 如果存在直接返回，不存在就进行读取配置文件，进行类加载的操作
// 具体的实现在 loadExtensionClasses 中
@SuppressWarnings("unchecked")
private T createExtension(String name) {
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    try {
        // 这里依然是缓存检查
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        // injectExtension 方法主要是对，生成的扩展点进行属性的注入
        // 如 RegistryProtocol 会在 injectExtension 时，对 cluster 属性进行赋值
        injectExtension(instance);
        // 这里使用 cachedWrapperClasses 包装类，对生成的实例进行包装
        // 比如 ReferenceConfig 中的代码 invoker = refprotocol.refer(interfaceClass, urls.get(0));
        // 这里 refer 方法生成的 invoker 实例是被包装之后的类，从而在客户卡调用的时候
        // 可以实现 mock，filter 等功能
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
            for (Class<?> wrapperClass : wrapperClasses) {
                // 循环包装类，进行包装
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("Extension instance(name: " + name + ", class: " +
                type + ")  could not be instantiated: " + t.getMessage(), t);
    }
}
```

### WrapperClass

`WrapperClass` 是一个典型的设计模式:`装饰器模式`

这里说明下`wrapperClasses`也是从`loadExtensionClasses`方法从`META-INF`读取配置文件而来

可以在 `META-INF/dubbo/internal/org.apache.dubbo.rpc.Protocol` 文件中看到下面的配置：

```properties
filter=org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper
listener=org.apache.dubbo.rpc.protocol.ProtocolListenerWrapper
mock=org.apache.dubbo.rpc.support.MockProtocol
```

看下判断是否是包装类的方法:

```java
private boolean isWrapperClass(Class<?> clazz) {
    try {
        clazz.getConstructor(type);
        return true;
    } catch (NoSuchMethodException e) {
        return false;
    }
}
// 上面代码的含义，只要提供了包含 SPI clazz (如:protocol) 这个参数的构造方法
// dubbo 就认为它是一个包装类(装饰器模式)
// 如 ProtocolFilterWrapper 的构造方法如下
// ProtocolFilterWrapper 就会当做包装类对 DubboProtocol 等进行包装
// 返回 ProtocolFilterWrapper 对象,而ProtocolFilterWrapper 的属性 protocol 是 DubboProtocol
public ProtocolFilterWrapper(Protocol protocol) {
    if (protocol == null) {
        throw new IllegalArgumentException("protocol == null");
    }
    this.protocol = protocol;
}
```

### injectExtension

```java
// injectExtension 负责解析当前类的所有set 方法
// 并从 objectFactory 中获取对象，进行属性的注入
private T injectExtension(T instance) {
try {
    if (objectFactory != null) {
        for (Method method : instance.getClass().getMethods()) {
            if (method.getName().startsWith("set")
                    && method.getParameterTypes().length == 1
                    && Modifier.isPublic(method.getModifiers())) {
                /**
                 * Check {@link DisableInject} to see if we need auto injection for this property
                 */
                if (method.getAnnotation(DisableInject.class) != null) {
                    continue;
                }
                Class<?> pt = method.getParameterTypes()[0];
                if (ReflectUtils.isPrimitives(pt)) {
                    continue;
                }
                try {
                    String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
                    // objectFactory 也是基于 SPI 加载的
                    Object object = objectFactory.getExtension(pt, property);
                    if (object != null) {
                        method.invoke(instance, object);
                    }
                } catch (Exception e) {
                    logger.error("fail to inject via method " + method.getName()
                            + " of interface " + type.getName() + ": " + e.getMessage(), e);
                }
            }
        }
    }
} catch (Exception e) {
    logger.error(e.getMessage(), e);
}
return instance;
}
```

### loadExtensionClasses

```java
// ExtensionLoader 中的静态变量
private static final String SERVICES_DIRECTORY = "META-INF/services/";
private static final String DUBBO_DIRECTORY = "META-INF/dubbo/";
private static final String DUBBO_INTERNAL_DIRECTORY = DUBBO_DIRECTORY + "internal/";
// synchronized in getExtensionClasses
private Map<String, Class<?>> loadExtensionClasses() {
    // 这里对扩展类上面的注解进行解析
    // 并赋值到 cachedDefaultName 上
    // 如果是 Protocol 那么cachedDefaultName=dubbo
    final SPI defaultAnnotation = type.getAnnotation(SPI.class);
    if (defaultAnnotation != null) {
        String value = defaultAnnotation.value();
        if ((value = value.trim()).length() > 0) {
            String[] names = NAME_SEPARATOR.split(value);
            if (names.length > 1) {
                throw new IllegalStateException("more than 1 default extension name on extension " + type.getName()
                        + ": " + Arrays.toString(names));
            }
            if (names.length == 1) {
                cachedDefaultName = names[0];
            }
        }
    }
    // 下面的几个方法，从 META-INF/services/ META-INF/dubbo/ META-INF/dubbo/internal/
    // 这三个目录下面去获取配置文件,进行解析
    // 把 org.apache 替换成 com.alibaba 是因为 dubbo 在apache 进行孵化，修改过包名
    Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
    loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName());
    loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName());
    loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName());
    loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    return extensionClasses;
}
```

### getAdaptiveExtension

`getAdaptiveExtension()` 与 `getExtension(String name)` 执行逻辑类似,
不同的地方是`getAdaptiveExtension`返回的是`***$Adaptive`(如：`Protocol$Adaptive`)
而`Adaptive`负责调用`getExtension`,通过 `name` 获取具体的实现类。

而`getAdaptiveExtension` 中使用 `createAdaptiveExtensionClassCode`,生成一个`Adaptive`类

例子：

```java
private static final Protocol refprotocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```

上面的方法会最终返回下面的这个类`Protocol$Adaptive`,这个类是通过动态拼接而成的 [Protocol\$Adaptive](#protocoladaptive)

而`Protocol$Adaptive` 这个类是由 `createAdaptiveExtensionClassCode` 这个方法动态生成的对象

### createAdaptiveExtensionClassCode

`ExtensionLoader#createAdaptiveExtensionClassCode` 这个方法负责生成具体的 `***$Adaptive` 实例，如: `Protocol$Adaptive`
此方法中使用了`javassist`等字节码技术，动态的创建`***$Adaptive`类。

这里的代码逻辑就是对 `org.apache.dubbo.rpc.Protocol` 接口中所有方法进行解析，生成一个实现类，包含接口中的所有方法

类似的 SPI 类还有（未全部包含）:

- Cluster\$Adaptive
- Dispatcher\$Adaptive
- Protocol\$Adaptive
- ProxyFactory\$Adaptive
- RegistryFactory\$Adaptive
- ThreadPool\$Adaptive
- Transporter\$Adaptive
- Validation\$Adaptive

`createAdaptiveExtensionClassCode` 生成的代码类内容，可以参考这里 [dubbo adaptive class](https://github.com/web1992/dubbos/tree/master/dubbo-source-code/src/main/java/cn/web1992)

## Protocol

- `ServiceConfig`中`protocol.export`这个方法的主要二个作用：

  - 启动一个本地的 TCP 服务（如：NettyServer 服务）
  - 把本地启动的 TCP 服务通过 Registry 类注册到注册中心 (如:`zookeeper`)

`ServiceConfig` 的初始化过程可以看这篇文章[dubbo provider init](dubbo-init.md#provider-init)

看一段来自 `org.apache.dubbo.config.ServiceConfig`  中的源码注释

```java
/**
 * The {@link Protocol} implementation with adaptive functionality,it will be different in different scenarios.
 * A particular {@link Protocol} implementation is determined by the protocol attribute in the {@link URL}.
 * For example:
 *
 * <li>when the url is registry://224.5.6.7:1234/org.apache.dubbo.registry.RegistryService?application=dubbo-sample,
 * then the protocol is <b>RegistryProtocol</b></li>
 *
 * <li>when the url is dubbo://224.5.6.7:1234/org.apache.dubbo.config.api.DemoService?application=dubbo-sample, then
 * the protocol is <b>DubboProtocol</b></li>
 * <p>
 * Actually，when the {@link ExtensionLoader} init the {@link Protocol} instants,it will automatically wraps two
 * layers, and eventually will get a <b>ProtocolFilterWrapper</b> or <b>ProtocolListenerWrapper</b>
 */
private static final Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```

`Protocol` 实现了接口自适应功能（自适应：根据参数的不同来选择不同的实现类进行调用）

如这个方法：

```java
 Exporter<?> exporter = protocol.export(wrapperInvoker);
```

`wrapperInvoker`是一个`org.apache.dubbo.rpc.Invoker`对象，Invoker 继承了 `org.apache.dubbo.common.Node`类
Node 类中有`URL getUrl();` 这个方法可以返回 URL 对象.

一个例子：如果 URL 中参数是`registry`的时候，调用的实现就是`RegistryProtocol`的`export`
如果 URL 中参数是`dubbo`的时候，调用的实现就是`DubboProtocol`的`export`

而实现这个功能的类就是`Protocol\$Adaptive`

## Protocol\$Adaptive

`ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();`这个返回的是包装之后的`Protocol` 类实例

具体的代码实现逻辑如下：

```java
import org.apache.dubbo.common.extension.ExtensionLoader;
// Protocol$Adaptive 是 Protocol 接口实现自适应类的关键
// 当我们执行 protocol.export 方法时，其实调用的是 Protocol$Adaptive 中的 export 方法
// Protocol$Adaptive 会根据 url 中的参数，或者 Invoker 中 url 的参数，通过 SPI 去加载具体的实现类
// 从而实现参数不同，调用不同的实现类，实现 dubbo 的(方法)自适应
public class Protocol$Adaptive implements org.apache.dubbo.rpc.Protocol {
    public void destroy() {
        throw new UnsupportedOperationException("method public abstract void org.apache.dubbo.rpc.Protocol.destroy() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
    }

    public int getDefaultPort() {
        throw new UnsupportedOperationException("method public abstract int org.apache.dubbo.rpc.Protocol.getDefaultPort() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
    }

    public org.apache.dubbo.rpc.Exporter export(org.apache.dubbo.rpc.Invoker arg0) throws org.apache.dubbo.rpc.RpcException {
        if (arg0 == null) {
            throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
        }
        if (arg0.getUrl() == null) {
            throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");
        }
        org.apache.dubbo.common.URL url = arg0.getUrl();
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null) {
            throw new IllegalStateException("Fail to get extension(org.apache.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        }
        org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.export(arg0);
    }

    public org.apache.dubbo.rpc.Invoker refer(java.lang.Class arg0, org.apache.dubbo.common.URL arg1) throws org.apache.dubbo.rpc.RpcException {
        if (arg1 == null) {
            throw new IllegalArgumentException("url == null");
        }
        org.apache.dubbo.common.URL url = arg1;
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null) {
            throw new IllegalStateException("Fail to get extension(org.apache.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        }
        org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.refer(arg0, arg1);
    }
}
```

可以看到`export`这个方法有一个参数`org.apache.dubbo.rpc.Invoker`，从`getUrl()`获取一个`org.apache.dubbo.common.URL`对象

然后获取`protocol`，如果为空，默认是`dubbo`，然后通过 `ExtensionLoader.getExtensionLoader.getExtension` 去加载具体的实现类

## Summary

1. dubbo 中通过 SPI 机制可以实现自定义扩展类，方便增加新功能
2. dubbo 中通过 ***$Adaptive 自适应类，通过 `URL` 参数的不同，实现不同的业务,更灵活，缺点难以 debug
3. dubbo 中通过 SPI 机制加载的类，会进行包装，属性注入，增强类的功能，提供 Filter,Moniter 等功能
4. 未完待续...
