[toc]

## Apache BookKeeper之MetaData管理

#### MetaData信息
这里主要有两类
* 有效的Bookie的列表 
 1. 用来跟踪哪些Bookie是有效的
 
* Ledger的相关信息
其相关操作如下:
 1. **createLedger**: 创建一个新的Ledger, 它拥有一个唯一ID和当前的Version(对应到zk的话，这个version就是znode的dataVersion);
 2. **removeLedgerMetadata**: 移除一个Ledger, 需要提供当前本地保存的Version, 和 MetaData Storage中的Version作check, 一致才允许作remove操作;
 3. **readLedgerMetadata**: 读取一个Ledger的相关meta信息, 同时需要更新此Ledger的meta信息的Version;
 4. **writeLedgerMetadata**: 更新Ledger的相关meta信息，同样需要提供当前本地保存的Version, 和 MetaData Storage中的Version作check, 一致才允许操作;
 5. **asyncProcessLedgers**: 遍历当前所有的Ledger，分别对其应用一个给定的处理函数;

#### MetaData Storage的选取
* 需要首先满足以下几点要求：
 1. 支持CAS操作： Check and Set, 比如上面提到的在删除和更新操作时需先比较Version;
 2. 针对连续write的优化;
 3. 针对Scan操作的优化;

* 目前来看合适的MetaData Storage有zookeeper, etcd, 如果ledger数量超级大，还可以使用HBase;
* Apache BookKeeper当前默认使用Zookeeper实现;

#### MetaData操作的实现
###### MetadataBookieDriver
在Apache BookKeeper中对MetaData的所有操作都被封装到一个抽象接口`MetadataBookieDriver`中;
```
public interface MetadataBookieDriver extends AutoCloseable {
    // 初始化当前的Driver
    MetadataBookieDriver initialize(ServerConfiguration conf,
                                    RegistrationListener listener,
                                    StatsLogger statsLogger)
        throws MetadataException;
    String getScheme();

    // RegistrationManager负责管理Bookie注册到Storage的相关操作
    RegistrationManager getRegistrationManager();

    LedgerManagerFactory getLedgerManagerFactory()
        throws MetadataException;

    LayoutManager getLayoutManager();

    @Override
    void close();
}
```

###### MetadataDrivers
负责管理所有的MetadataBookieDriver
* 将所有Driver信息保存在 `private static final ConcurrentMap<String, MetadataBookieDriverInfo> bookieDrivers;`, 其中key是scheme, value是`MetadataBookieDriverInfo`, 定义如下:
```
static class MetadataBookieDriverInfo {
        final Class<? extends MetadataBookieDriver> driverClass;
        final String driverClassName;

        MetadataBookieDriverInfo(Class<? extends MetadataBookieDriver> driverClass) {
            this.driverClass = driverClass;
            this.driverClassName = driverClass.getName();
        }
    }
```
利用java的反射机制根据`driverClass`即可产生出对应的`MetadataBookieDriver`对象;
* 默认包含`org.apache.bookkeeper.meta.zk.ZKMetadataBookieDriver`, 即`ZkMetadataBookieDriver`, 其scheme为`zk`
* 获取MetadataBookieDriver
```
public static MetadataBookieDriver getBookieDriver(URI uri) {
        //对于zk来说，这个uri形如:zk+hierarchical://10.1.1.1:2181;10.1.1.2:2181;10.1.1.3:2181/ledgers
        String scheme = uri.getScheme();
		// scheme 为 zk
        scheme = scheme.toLowerCase();
        String[] schemeParts = StringUtils.split(scheme, '+');
		
        if (!initialized) {
            initialize();
        }
		
        MetadataBookieDriverInfo driverInfo = bookieDrivers.get(scheme.toLowerCase());
        if (null == driverInfo) {
            throw new IllegalArgumentException("Unknown backend " + scheme);
        }
		// 利用java的反射机制 
        return ReflectionUtils.newInstance(driverInfo.driverClass);
    }
```

###### ZkMetadataBookieDriver的实现
* `getRegistrationManager`: 返回`ZKRegistrationManager`
```
 if (null == regManager) {
            regManager = new ZKRegistrationManager(
                serverConf,
                zk,
                listener
            );
        }
        return regManager;
```
* `initialize`: 主要是调用其父类`ZKMetadataDriverBase`的`initialize`方法
主要作的事情就是创建了操作zk的`Zookeeper`对象和`ZkLayoutManager`对象
```
    protected void initialize(AbstractConfiguration<?> conf,
                              StatsLogger statsLogger,
                              RetryPolicy zkRetryPolicy,
                              Optional<Object> optionalCtx) throws MetadataException {
        this.conf = conf;
        this.acls = ZkUtils.getACLs(conf);

        if (optionalCtx.isPresent()
         ... 
        } else {
            final String metadataServiceUriStr;
            try {
                metadataServiceUriStr = conf.getMetadataServiceUri();
            } catch (ConfigurationException e) {
                throw new MetadataException(
                    Code.INVALID_METADATA_SERVICE_URI, e);
            }
            ...
            final String zkServers = getZKServersFromServiceUri(metadataServiceUri);
            try {
                this.zk = ZooKeeperClient.newBuilder()
                    .connectString(zkServers)
                    .sessionTimeoutMs(conf.getZkTimeout())
                    .operationRetryPolicy(zkRetryPolicy)
                    .requestRateLimit(conf.getZkRequestRateLimit())
                    .statsLogger(statsLogger)
                    .build();

                if (null == zk.exists(bookieReadonlyRegistrationPath, false)) {
                    try {
                        zk.create(bookieReadonlyRegistrationPath,
                            EMPTY_BYTE_ARRAY,
                            acls,
                            CreateMode.PERSISTENT);
                    } catch (KeeperException.NodeExistsException e) {
                    } catch (KeeperException.NoNodeException e) {
                    }
                }
            } catch (IOException | KeeperException e) {
                throw me;
            }
            this.ownZKHandle = true;
        }

        // once created the zookeeper client, create the layout manager and registration client
        this.layoutManager = new ZkLayoutManager(
            zk,
            ledgersRootPath,
            acls);
    }
```
* `getLedgerManagerFactory`: 直接沿用其父类`ZKMetadataDriverBase`的, 返回`LedgerManagerFactory`对象，用于创建`LedgerManager`
```
 public synchronized LedgerManagerFactory getLedgerManagerFactory()
            throws MetadataException {
        if (null == lmFactory) {
            try {
                lmFactory = AbstractZkLedgerManagerFactory.newLedgerManagerFactory(
                    conf,
                    layoutManager);
            } catch (IOException e) {
                throw new MetadataException(
                    Code.METADATA_SERVICE_ERROR, "Failed to initialized ledger manager factory", e);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw e;
            }
        }
        return lmFactory;
    }
```

###### `ZkRegistrationManager`
* 主要用于当前bookie信息以临时节点的方式注册到zk上，取消注册，写cookie, 读取cookie
* 注册bookie实现 `doRegisterBookie`: 在zk上创建临时节点
```
 private void doRegisterBookie(String regPath) throws BookieException {
        // ZK ephemeral node for this Bookie.
        try {
            if (!checkRegNodeAndWaitExpired(regPath)) {
                // Create the ZK ephemeral node for this Bookie.
                zk.create(regPath, new byte[0], zkAcls, CreateMode.EPHEMERAL);
                zkRegManagerInitialized = true;
            }
        } catch (KeeperException ke) {
            throw new MetadataStoreException(ke);
        } catch (InterruptedException ie) {
            throw new MetadataStoreException(ie);
        } catch (IOException e) {
            throw new MetadataStoreException(e);
        }
    }
```
* 写cookie操作 `writeCookie`: 在每个bookie的cookie信息写在形如`/ledgers/cookies/10.209.1.1:3181`的节点, cookie内容形如:
```
4  ----  当前cookie的layout版本号
bookieHost: "10.209.240.36:3181"
journalDir: "/data/bookkeeper/journal"
ledgerDirs: "1\t/data/bookkeeper/ledger"
instanceId: "eb314bf8-885e-4c60-803d-32fd7858d790" ---- 当前集群的id
```
* 初始化新的cluster `initNewCluster`:
创建 /ledgers 节点
创建 /ledgers/available 节点
创建 /ledgers/available/readonly 节点
创建 /ledgers/INSTANCEID 节点
创建新的 LedgerManagerFactory
```
public boolean initNewCluster() throws Exception {
        String zkServers = ZKMetadataDriverBase.resolveZkServers(conf);
        String instanceIdPath = ledgersRootPath + "/" + INSTANCEID;

        boolean ledgerRootExists = null != zk.exists(ledgersRootPath, false);

        if (ledgerRootExists) {
            return false;
        }

        List<Op> multiOps = Lists.newArrayListWithExpectedSize(4);
        // Create ledgers root node
        multiOps.add(Op.create(ledgersRootPath, EMPTY_BYTE_ARRAY, zkAcls, CreateMode.PERSISTENT));

        // create available bookies node
        multiOps.add(Op.create(bookieRegistrationPath, EMPTY_BYTE_ARRAY, zkAcls, CreateMode.PERSISTENT));

        // create readonly bookies node
        multiOps.add(Op.create(
            bookieReadonlyRegistrationPath,
            EMPTY_BYTE_ARRAY,
            zkAcls,
            CreateMode.PERSISTENT));

        // create INSTANCEID
        String instanceId = UUID.randomUUID().toString();
        multiOps.add(Op.create(instanceIdPath, instanceId.getBytes(UTF_8),
                zkAcls, CreateMode.PERSISTENT));

        // execute the multi ops
        // 这个multi操作组合了对多个node的操作，本质上也是原子操作，要么都成功，要么都失败
        zk.multi(multiOps);

        // creates the new layout and stores in zookeeper
		// 如果当前 zk上/ledger/LAYOUT节点没有数据，且layoutManager不为null, 下面这个调用会写入新的/ledger/LAYOUT数据
        AbstractZkLedgerManagerFactory.newLedgerManagerFactory(conf, layoutManager);
        return true;
    }
```

###### LedgerManagerFactory
* 前面我们已经说过存储在zk上的meta信息，其中最主要的一个就是ledger的信息，ledger的数量可能很少也可能很多，都存储在zk上的话，需要有个合理的组织形式，目前主要有两种：

 1. **Flat Ledger Layout**: 所有的ledger信息都存储在唯一的一个znode（比如/ledger）下，这些ledger节点的命名以"L"开头，后面是它的id,形如"/ledger/L001"；这样的存储有一个问题，如果ledger数据太多的话，通过zk的getChilds接口获取所有的ledger时，返回的结果会超过zk的package size，从而获取失败;
 2. **Hierarchical ledger manager**: 分层存储，先利用zk的`EPHEMERAL_SEQUENTIAL znode`产生一个全局唯一的ledger id, 这种方式产生的id有10位，形如`0000000001`, 将其拆成两层 `/ledger/00/0000/L0001`，作为一个znode,存储相对应的ledger信息;
 3. **LongHierarchical ledger manager**: 上面的ledger id是31位，这个是63位, 在zk上的表示形如 `/ledger/000/0000/0000/0000/L0001`
* `LedgerManagerFactory`: 创建LedgerManager,其继承关系为下
![ledger-manager-factory-classe](file:///home/lw/docs/apache bookkeeper/ledger-factory-classes1.png)
 1. `format`接口： 删除zk上所有的ledger信息，删除/ledger/LAYOUT信息，写入新的layout信息
```
public void format(AbstractConfiguration<?> conf, LayoutManager layoutManager)
            throws InterruptedException, KeeperException, IOException {
        try (AbstractZkLedgerManager ledgerManager = (AbstractZkLedgerManager) newLedgerManager()) {
            String ledgersRootPath = ZKMetadataDriverBase.resolveZkLedgersRootPath(conf);
            List<String> children = zk.getChildren(ledgersRootPath, false);
            for (String child : children) {
              // 采用 hierarchical layou时，ledger信息是在zk的形如 /ledger/00的znode下,下面的代码就是删除所有的ledger信息
                if (!AbstractZkLedgerManager.isSpecialZnode(child) && ledgerManager.isLedgerParentNode(child)) {
                    ZKUtil.deleteRecursive(zk, ledgersRootPath + "/" + child);
                }
            }
        }

        Class<? extends LedgerManagerFactory> factoryClass;
        try {
            factoryClass = conf.getLedgerManagerFactoryClass();
        } catch (ConfigurationException e) {
            throw new IOException("Failed to get ledger manager factory class from configuration : ", e);
        }

        // 删除zk上的 /ledger/LAYOUT
        layoutManager.deleteLedgerLayout();
        // Create new layout information again.
        // 将新的layout写到zk的 /ledger/LAYOUT下
        createNewLMFactory(conf, layoutManager, factoryClass);
    }
```
 2. `validateAndNukeExistingCluster`: 清除zk上的所有节点 
 3. `newLedgerIdGenerator`: 返回一个ledger id的产生器:
```
public LedgerIdGenerator newLedgerIdGenerator() {
        List<ACL> zkAcls = ZkUtils.getACLs(conf);
        String zkLedgersRootPath = ZKMetadataDriverBase.resolveZkLedgersRootPath(conf);
        ZkLedgerIdGenerator subIdGenerator = new ZkLedgerIdGenerator(zk, zkLedgersRootPath,
                LegacyHierarchicalLedgerManager.IDGEN_ZNODE, zkAcls);
        return new LongZkLedgerIdGenerator(zk, zkLedgersRootPath, LongHierarchicalLedgerManager.IDGEN_ZNODE,
                subIdGenerator, zkAcls);
    }
```
支持产生31位和64位的id, 目前看起来足够使用了。具体实现这里不讲了，大家可以看下源码，都是借助于zk的`EPHEMERAL_SEQUENTIAL znode`;
 4. `newLedgerManager`：创建Ledgermanager对象
```
public LedgerManager newLedgerManager() {
        return new HierarchicalLedgerManager(conf, zk);
    }
```

###### LedgerManager
* 先看一下类的层级关系和需要实现的接口函数
 ![ledger-manager-classe](file:///home/lw/docs/apache bookkeeper/ledgermanager-hi.png)
* `createLedgerMetadata`: 异步创建新的Ledger， 返回 `CompletableFuture<...>`,
Metadata version大于2时，ledger metadata中需添加ctoken
```
    public CompletableFuture<Versioned<LedgerMetadata>> createLedgerMetadata(long ledgerId,
                                                                             LedgerMetadata inputMetadata) {
        CompletableFuture<Versioned<LedgerMetadata>> promise = new CompletableFuture<>();
        /*
         * Metadata version大于2时，ledger metadata中需添加ctoken
         */
        final long cToken = ThreadLocalRandom.current().nextLong(Long.MAX_VALUE);
        final LedgerMetadata metadata;
        if (inputMetadata.getMetadataFormatVersion() > LedgerMetadataSerDe.METADATA_FORMAT_VERSION_2) {
            metadata = LedgerMetadataBuilder.from(inputMetadata).withCToken(cToken).build();
        } else {
            metadata = inputMetadata;
        }
        String ledgerPath = getLedgerPath(ledgerId);
		
		// 这个scb是zk操作完后的回调函数
        StringCallback scb = new StringCallback() {
            @Override
            public void processResult(int rc, String path, Object ctx, String name) {
                if (rc == Code.OK.intValue()) {
				// 创建ledger成功
                    promise.complete(new Versioned<>(metadata, new LongVersion(0)));
                } else if (rc == Code.NODEEXISTS.intValue()) {
				// 处理创建的ledger节点已经存在的情况
                    if (metadata.getMetadataFormatVersion() > 2) {
					//读取当前已有的ledger的meta信息
                        CompletableFuture<Versioned<LedgerMetadata>> readFuture = readLedgerMetadata(ledgerId);
                        readFuture.handle((readMetadata, exception) -> {
                            if (exception == null) {
							// 利用这个ctoken来判断是不是当前的操作
                                if (readMetadata.getValue().getCToken() == cToken) {
                                    FutureUtils.complete(promise, new Versioned<>(metadata, new LongVersion(0)));
                                } else {
                                    promise.completeExceptionally(new BKException.BKLedgerExistException());
                                }
                            } else if (exception instanceof KeeperException.NoNodeException) {
                                promise.completeExceptionally(new BKException.BKLedgerExistException());
                            } else {
                                promise.completeExceptionally(new BKException.ZKException());
                            }
                            return null;
                        });
                    } else {
                        promise.completeExceptionally(new BKException.BKLedgerExistException());
                    }
                } else {
                    promise.completeExceptionally(new BKException.ZKException());
                }
            }
        };
        final byte[] data;
        try {
            data = serDe.serialize(metadata);
        } catch (IOException ioe) {
            promise.completeExceptionally(new BKException.BKMetadataSerializationException(ioe));
            return promise;
        }

        List<ACL> zkAcls = ZkUtils.getACLs(conf);
		// 异步创建ledger节点，如果其父节点不存在，会递归创建
        ZkUtils.asyncCreateFullPathOptimistic(zk, ledgerPath, data, zkAcls,
                                              CreateMode.PERSISTENT, scb, null);
        return promise;
    }
```
* `removeLedgerMetadata`: 异步删除ledger的meta信息，删除时不光提供ledger id，还要提供其在zk上的data version,供调用zk.delete时用
* `readLedgerMetadata`: 异步读取ledger的meta信息
```
protected CompletableFuture<Versioned<LedgerMetadata>> readLedgerMetadata(long ledgerId, Watcher watcher) {
        CompletableFuture<Versioned<LedgerMetadata>> promise = new CompletableFuture<>();
        zk.getData(getLedgerPath(ledgerId), watcher, new DataCallback() {
            @Override
            public void processResult(int rc, String path, Object ctx, byte[] data, Stat stat) {
                if (rc == KeeperException.Code.NONODE.intValue()) {
                    promise.completeExceptionally(new BKException.BKNoSuchLedgerExistsOnMetadataServerException());
                    return;
                }
                if (rc != KeeperException.Code.OK.intValue()) {
                    promise.completeExceptionally(new BKException.ZKException());
                    return;
                }
                if (stat == null) {
                    promise.completeExceptionally(new BKException.ZKException());
                    return;
                }

                try {
				// 构造 LedgerMetadata信息
                    LongVersion version = new LongVersion(stat.getVersion());
                    LedgerMetadata metadata = serDe.parseConfig(data, Optional.of(stat.getCtime()));
                    promise.complete(new Versioned<>(metadata, version));
                } catch (Throwable t) {
                    promise.completeExceptionally(new BKException.ZKException());
                }
            }
        }, null);
        return promise;
    }
```


