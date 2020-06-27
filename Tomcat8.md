### tomcat 为什么有 war 包

对于 java 程序员而言，xx.jar 往往意味着依赖，tomcat 通过 war 后缀判断是否是 web 应用

```java
protected void deployApps() {

    File appBase = host.getAppBaseFile();
    File configBase = host.getConfigBaseFile();
    String[] filteredAppPaths = filterAppPaths(appBase.list());
    // Deploy XML descriptors from configBase
    deployDescriptors(configBase, configBase.list());
    // Deploy WARs
    deployWARs(appBase, filteredAppPaths);
    // Deploy expanded folders
    deployDirectories(appBase, filteredAppPaths);

}
```

在里面的 deployWARs 函数，有以下代码

```java
for (Future<?> result : results) {
    try {
      result.get();
    } catch (Exception e) {
      log.error(sm.getString(
        "hostConfig.deployWar.threaded.error"), e);
    }
}
```

tomcat 是异步的去部署 webapps 目录下的所有项目的。

### Container 接口

有四个接口继承 Container 接口

Engine—>Host—>Context—>Wrapper—>Servlet

- Context
- Engine
- Host
- Wrapper

```java
PipeLine
	List<Valve> vales;

Engine
	PipeLine pipeline;
	List<Host>
	
Host
	PipeLine pipeline;
	List<Context>
	
Context
	PipeLine pipeline;
	List<Wrapper>
	
Wrapper
	PipeLine pipeline;
	List<Servlet> servlets;
```

### tomcat 模块分层

![image-20200624231341750](/Users/xmly/Documents/imgs/tomcat-modules.png)

### Service 包含内容

一个 Service 都包含着多个连接器组件 Connector(Coyote 实现) 和一个容器组件 Container。在 Tomcat 启动的时候，会初始化一个 catalina 的实例。