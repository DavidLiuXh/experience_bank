[toc]

## Apache Bookkeeper的启动流程解析 
#### 全局main函数实现
##### 启动代码
位于`bookkeeper-server/src/main/java/org/apache/bookkeeper/server/Main.java`中
```
public static void main(String[] args) {
        int retCode = doMain(args);
        Runtime.getRuntime().exit(retCode);
    }

    static int doMain(String[] args) {
        ServerConfigurtion conf;
        // 0. parse command line
        try {
            conf = parseCommandLine(args);
        } catch (IllegalArgumentException iae) {
            return ExitCode.INVALID_CONF;
        }

        // 1. building the component stack:
        LifecycleComponent server;
        try {
            server = buildBookieServer(new BookieConfiguration(conf));
        } catch (Exception e) {
            log.error("Failed to build bookie server", e);
            return ExitCode.SERVER_EXCEPTION;
        }

        // 2. start the server
        try {
            ComponentStarter.startComponent(server).get();
        } catch (InterruptedException ie) {
            Thread.currentThread().interrupt();
            // the server is interrupted
            log.info("Bookie server is interrupted. Exiting ...");
        } catch (ExecutionException ee) {
            log.error("Error in bookie shutdown", ee.getCause());
            return ExitCode.SERVER_EXCEPTION;
        }
        return ExitCode.OK;
    }
```
启动代码很简洁，且作了简要注释，主要启动流程都封装在了`LifecycleComponent`中;

##### 配置解析
* 目前所有的配置项都可以通过配置文件来设置，这也是推荐的方式。因此大部分情况下启动参数只需要传一个配置文件的路径即可;
* `private static ServerConfiguration parseCommandLine(String[] args) 主要是从配置文件里加载各种配置，最后生成`ServerConfiguration`对象，后续所有配置的读取和修改都通过这个对象来操作;

##### 服务构建
* 每个bookie包括若干个子服务，每个子服务对应一个`Component`, 需要一一启动和关闭，这里使用`LifecycleComponent`来管理所有子服务的生命周期;
* 我们来看一下`LifecycleComponet`:
 ```
 public class LifecycleComponentStack implements LifecycleComponent {
    public static Builder newBuilder() {
        return new Builder();
    }

    /**
     * Builder to build a stack of {@link LifecycleComponent}s.
     */
    public static class Builder {
	    ...
        private final List<LifecycleComponent> components;

        private Builder() {
            components = Lists.newArrayList();
        }

        public Builder addComponent(LifecycleComponent component) {
            checkNotNull(component, "Lifecycle component is null");
            components.add(component);
            return this;
        }

        public LifecycleComponentStack build() {
            checkNotNull(name, "Lifecycle component stack name is not provided");
            checkArgument(!components.isEmpty(), "Lifecycle component stack is empty : " + components);
            return new LifecycleComponentStack(
                name,
                ImmutableList.copyOf(components));
        }
    }

    private final String name;
    private final ImmutableList<LifecycleComponent> components;

    private LifecycleComponentStack(String name,
                                    ImmutableList<LifecycleComponent> components) {
        this.name = name;
        this.components = components;
    }
    ....
    @Override
    public void start() {
        components.forEach(component -> component.start());
    }

    @Override
    public void stop() {
        components.reverse().forEach(component -> component.stop());
    }

    @Override
    public void close() {
        components.reverse().forEach(component -> component.close());
    }
}
 ```
 1. 首先它内置了`Builder`类，通过其`addComponent`方法将子组件加入进来，然后通过其`build`方法生成`LifecycleComponent`;
 2. 从定义上来看`LifecycleComponent`也实现了`LifecycleComponent`,因此它也实现了`start` `stop` `close`, 只不过这些方    法操作的是其包含的合部Component;
 3. 构建启动所需的所有子服务，实现在`buildBookieServer`中
 ```
 public static LifecycleComponentStack buildBookieServer(BookieConfiguration conf) throws Exception {
        LifecycleComponentStack.Builder serverBuilder = LifecycleComponentStack.newBuilder().withName("bookie-server");

        // 1. build stats provider
        serverBuilder.addComponent(statsProviderService);

        // 2. build bookie server
        serverBuilder.addComponent(bookieService);

        if (conf.getServerConf().isLocalScrubEnabled()) {
            serverBuilder.addComponent(
                    new ScrubberService(
                            rootStatsLogger.scope(ScrubberStats.SCOPE),
                    conf, bookieService.getServer().getBookie().getLedgerStorage()));
        }

        // 3. build auto recovery
            serverBuilder.addComponent(autoRecoveryService);
            log.info("Load lifecycle component : {}", AutoRecoveryService.class.getName());

        // 4. build http service
            serverBuilder.addComponent(httpService);

        // 5. build extra services
		...
		
        return serverBuilder.build();
    }
 ```
 这里需要启动组件主要有：StatsProviderService, BookieService, ScrubberService, AutoRecoverryService和HttpService
 
##### 服务启动
* 组件启动使用`ComponentStarter`来实现
```
public static CompletableFuture<Void> startComponent(LifecycleComponent component) {
        CompletableFuture<Void> future = new CompletableFuture<>();
        final Thread shutdownHookThread = new Thread(
            new ComponentShutdownHook(component, future),
            "component-shutdown-thread"
        );

        // register a shutdown hook
        Runtime.getRuntime().addShutdownHook(shutdownHookThread);

        // register a component exception handler
        component.setExceptionHandler((t, e) -> {
            // start the shutdown hook when an uncaught exception happen in the lifecycle component.
            shutdownHookThread.start();
        });

        component.start();
        return future;
    }
```
这个函数返回`CompletableFuture<Void> future`，当整个bookie结束或者抛出了未捕获的异常时，这个`future`将被complete，对应`doMain`中的代码就是
```
    try {
            ComponentStarter.startComponent(server).get();
        } catch (InterruptedException ie) {
            Thread.currentThread().interrupt();
            // the server is interrupted
            log.info("Bookie server is interrupted. Exiting ...");
        } catch (ExecutionException ee) {
            log.error("Error in bookie shutdown", ee.getCause());
            return ExitCode.SERVER_EXCEPTION;
        }
```
其中`ComponentStarter.startComponent(server).get()`将阻塞等待future完成