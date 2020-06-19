* 编译针对当前 github上influxdb的[master](https://github.com/influxdata/influxdb)代码
* 其实github上的[CONTRIBUTING.md](https://github.com/influxdata/influxdb/blob/master/CONTRIBUTING.md) 里已经说的很明白,按其一步步来开即开，唯一遇到的问题可能就是下载依赖时被墙无法下载，下文给了解决方案;
* 我们按[CONTRIBUTING.md](https://github.com/influxdata/influxdb/blob/master/CONTRIBUTING.md) 上的步骤再来梳理一下
1. **安装golang 1.11**, 最新版 Influxdb编译要求golang 1.11的支持,这个大家各显神通吧,安装好后设置好你的`GOPATH`;
2. **安装Dep**, 这个用来下载编译依赖用，针对被墙的依赖，这个并没有什么用;
`go get github.com/golang/dep/cmd/dep`;
安装好后`dep`在你的`$GOPATH/bin`下;
3. **git clone github上的Influxdb代码**：
3.1 在你的`$GOPATH`目录下建立目录`github.com/influxdata`;
3.2 进入到目录`$GOPATH/github.com/influxdata`下，执行`git clone https://github.com/influxdata/influxdb.git`;
4. **下载依赖**:
4.1 进入到目录`$GOPATH/github.com/influxdata/influxdb`下, 执行`$GOPATH/bin/dep ensure`,不出意外的话，应该有很多无法下载，怎么办？往下看
4.2 在Influxdb源码下有个列出了所有依赖的文件[DEPENDENCIES.md](https://raw.githubusercontent.com/influxdata/influxdb/master/DEPENDENCIES.md),上面的`dep ensure`无法下载的应该都是类似`golang.org/x/time`这种从`golang.org`下载的，但其实它们在github上也都有对应的下载地址，我们可以手动下载，比如说针对这个`golang.org/x/time`:
 a. 首先 `go get github.com/x/time`，会将其下载到`$GOPATH/github.com/x/time`下
 b. 再将 `$GOPATH/github.com/x/time` 移动到 `$GOPATH/golang.org/x/time`下
4.3 如果你不想手动下载，我这里提供一个打包好的，里面是完整的包括influxdb源码和其依赖， 下载链接: https://pan.baidu.com/s/1O7g74-bdyRyy0a_erWUFwA 提取码: shrw
5. **编译**：
5.1 进入到目录`$GOPATH/github.com/influxdata/influxdb`;
5.2 `go clean ./...`
5.3 `go install ./...`
5.4 编译成功后，要以在`$GOPATH/bin`下找到编译好的可执行文件
 