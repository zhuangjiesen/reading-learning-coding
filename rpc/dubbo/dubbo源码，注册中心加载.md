# dubbo源码，注册中心加载


### 源码入口：
```
Spring 解析配置类 DubboNamespaceHandler.java ，每个配置对应一个bean
```

### 加载配置
```
每个<dubbo:service /> 标签，对应一个ServiceBean ，它是个bean 并且实现接口 InitializingBean.java
就会触发 afterPropertiesSet() 方法
```

### 加载链路：

```
- ServiceBean.afterPropertiesSet()
- ServiceConfig.export()
- ServiceConfig.doExport()
- ServiceConfig.doExportUrls()
- AbstractInterfaceConfig.loadRegistries(boolean provider)
- 代码段 ExtensionLoader.getExtensionLoader(RegistryFactory.class).hasExtension("remote") ， 加载注册中心工厂
- ExtensionLoader.loadExtensionClasses() 加载spi拓展接口，加载目录下的接口文件


```


```

   // synchronized in getExtensionClasses
    private Map<String, Class<?>> loadExtensionClasses() {
        final SPI defaultAnnotation = type.getAnnotation(SPI.class);
        if (defaultAnnotation != null) {
            String value = defaultAnnotation.value();
            if ((value = value.trim()).length() > 0) {
                String[] names = NAME_SEPARATOR.split(value);
                if (names.length > 1) {
                    throw new IllegalStateException("more than 1 default extension name on extension " + type.getName()
                            + ": " + Arrays.toString(names));
                }
                if (names.length == 1) cachedDefaultName = names[0];
            }
        }

        Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
        loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
        loadDirectory(extensionClasses, DUBBO_DIRECTORY);
        loadDirectory(extensionClasses, SERVICES_DIRECTORY);
        return extensionClasses;
    }
```

### 核心

```
如果按自己的思路，注册中心就是提供一个缓存服务和变更通知服务，
每次调用前，
需要通过注册中心获取url(或者走缓存)，
只要变更的时候修改的url，
就能保持调用链路的架构优雅

注册中心不直接跟invoker层或者protocol层交互
Registry.java 最后是封装成了 Directory.java 类，对外提供url调用参数
Invoker.java 的子类AbstractClusterInvoker.java中有属性 directory
在select() 方法中，会调用 getUrl() 获取url
然后调用 doSelect() 方法可以获取到DubboInvoker.java对象，远程调用

```

