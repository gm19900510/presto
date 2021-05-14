# 简介
Presto是完全基于内存的并行计算以及分布式SQL交互式查询引擎。它可以共享Hive的元数据，然后直接访问HDFS中的数据。同Impala一样，作为Hadoop之上的SQL交互式查询引擎，通常比Hive要快5-10倍。Presto是一个运行在多台服务器上的分布式系统。完整安装包括一个coordinator和多个worker。

# 一、准备工作
### 1、完成Hadoop集群和Hive安装
可参照：[https://gaoming.blog.csdn.net/article/details/107399914](https://gaoming.blog.csdn.net/article/details/107399914) 利用CM离线安装CDH。
### 2、安装包下载
下载地址：[https://repo1.maven.org/maven2/com/facebook/presto/presto-server/](https://repo1.maven.org/maven2/com/facebook/presto/presto-server/) ，我选择的是0.232版本。

```bash
# 服务
wget -t 0 -c https://repo1.maven.org/maven2/com/facebook/presto/presto-server/0.232/presto-server-0.232.tar.gz
# JDBC驱动
wget -c -t 0 https://repo1.maven.org/maven2/com/facebook/presto/presto-jdbc/0.232/presto-jdbc-0.232.jar
# 命令行工具
wget -t 0 -c https://repo1.maven.org/maven2/com/facebook/presto/presto-cli/0.232/presto-cli-0.232-executable.jar
```
### 3、服务角色说明
服务器IP     | 名称
-------- | -----
192.168.3. 197 | S0
192.168.3. 18  | S1
192.168.3. 161  | S2
192.168.3. 136  | S3
由于小编的CDH集群是四个节点，那么此处也配置四个节点，其中S3节点安装coordinator，S0、S1、S2节点安装worker。
### 4、文件传输及解压
（1）文件上传至`/opt/cloudera/parcels`，并解压

```bash
tar zxvf presto-server-0.232.tar.gz -C /opt/cloudera/parcels
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200824104556322.png#pic_left)
（2）将解压好的目录分发到其余三个节点

```bash
scp -r /opt/cloudera/parcels/presto-server-0.232 S0:/opt/cloudera/parcels
scp -r /opt/cloudera/parcels/presto-server-0.232 S1:/opt/cloudera/parcels
scp -r /opt/cloudera/parcels/presto-server-0.232 S2:/opt/cloudera/parcels
```

（3）为`所有节点`的presto-server-0.232目录创建软连接

```bash
ln -s presto-server-0.232 presto
```

> 备注：此处或许对jdk的版本要求比严，jdk小版本尽量大于151。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200824104932316.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70#pic_center)

# 二、准备Presto的配置文件
### 1、创建etc目录
`所有节点`的`/opt/cloudera/parcels/presto`目录下创建`etc`目录，用于存放配置文件。

```bash
mkdir etc
```
### 2、新建node.properties文件
所有节点新建`node.properties`文件,内容如下（这里以S3节点为例，其他节点需要更改node.id等内容）：

```bash
node.environment=production
node.id=presto-s3
node.data-dir=/data/presto
```

>node.environment是集群名称。所有在同一个集群中的Presto节点必须拥有相同的集群名称。

>node.id是每个Presto节点的唯一标识。
每个节点的node.id都必须是唯一的。
在Presto进行重启或者升级过程中每个节点的node.id必须保持不变。
如果在一个节点上安装多个Presto实例（例如：在同一台机器上安装多个Presto节点），那么每个Presto节点必须拥有唯一的node.id。

>node.data-dir：数据存储目录的位置（操作系统上的路径）。Presto将会把日期和数据存储在这个目录下。
### 3、新建jvm.config文件
`所有节点`新建`jvm.config`文件，配置JVM参数，内容如下：

```go
-server
-Xmx12G
-XX:+UseG1GC
-XX:G1HeapRegionSize=32M
-XX:+UseGCOverheadLimit
-XX:+ExplicitGCInvokesConcurrent
-XX:+HeapDumpOnOutOfMemoryError
-XX:+ExitOnOutOfMemoryError
```
### 4、新建config.properties文件
`所有节点`创建`config.properties`文件，这里需要注意的是`coordinator（S3）`节点与`worker（S0、S1、S3）`节点的内容不一样

（1）`coordinator（S3）`节点的配置如下：

```bash
# 允许这个Presto实例充当协调器;true:协调者,false:表示workers
coordinator=true
# 否允许在coordinator服务中进行调度工作。对于大型的集群，在一个节点上的Presto server即作为coordinator又作为worker将会降低查询性能。因为如果一个服务器作为worker使用，大部分的资源都会被worker占用，就不会有足够的资源进行关键任务调度、管理和监控查询执行。
node-scheduler.include-coordinator=false
# 指定HTTP服务器的端口。Presto使用HTTP进行所有内部和外部通信。
http-server.http.port=10899
# 整个集群可以使用的最大用户执行内存
query.max-memory=24GB
# 每个机器上用于执行用户任务的内存大小
query.max-memory-per-node=6GB
# 每个节点上用于系统与用户任务的内存大小，该参数据包括上一个参数，多出系统所用内存，比如系统分配读写等，超出限制将kill
query.max-total-memory-per-node=8GB
# Presto使用发现服务查找集群中的所有节点。每个Presto实例将在启动时向DiscoveryService注册。为了简化部署和避免运行额外的服务，Presto协调器可以运行发现服务的嵌入式版本。它与Presto共享HTTP服务器，因此使用相同的端口。
discovery-server.enabled=true
# 访问地址;发现号服务器的URI
discovery.uri=http://S3:10899
```
（2）`worker（S0、S1、S3）`节点的配置如下：

```bash
coordinator=false
http-server.http.port=10899
query.max-memory-per-node=6GB
query.max-total-memory-per-node=8GB
discovery.uri=http://S3:10899
```
### 5、新建日志文件log.properties
所有节点新建日志文件`log.properties`，内容如下：

```bash
com.facebook.presto=INFO
```
# 三、Presto服务的启动和停止

```bash
# 启动
/opt/cloudera/parcels/presto/bin/launcher start
# 停止
/opt/cloudera/parcels/presto/bin/launcher stop
# 重启
/opt/cloudera/parcels/presto/bin/launcher restart
```
访问`coordinator（S3）`节点配置的地址：[http://s3:10899](http://s3:10899)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200824110644335.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70#pic_left)
关于Presto的更多命令，可以通过如下命令查看：

```go
/opt/cloudera/parcels/presto/bin/launcher --help
```
# 四、Presto集成Hive
### 1、新建catalog目录
在`所有节点`的`/opt/cloudera/parcels/presto/etc`目录下新建`catalog`目录：

```bash
mkdir catalog
```
### 2、创建hive.properties文件
所有节点在`catalog`目录下创建`hive.properties`文件，该文件与Hive服务集成使用，内容如下：

```bash
connector.name=hive-hadoop2
hive.metastore.uri=thrift://S0:9083
```

### 3、修改jvm.config文件
`所有节点`修改`presto的jvm.config`,在配置文件中增加Presto访问HDFS的用户名，如下所示：

```go
-server
-Xmx12G
-XX:+UseG1GC
-XX:G1HeapRegionSize=32M
-XX:+UseGCOverheadLimit
-XX:+ExplicitGCInvokesConcurrent
-XX:+HeapDumpOnOutOfMemoryError
-XX:+ExitOnOutOfMemoryError
-DHADOOP_USER_NAME=presto
```
### 4、创建用户
在`所有节点`创建`Presto访问HDFS的用户名`，上述用户为`presto`，需要新建此用户。
```go
useradd presto
```
### 5、重启presto服务
```bash
/opt/cloudera/parcels/presto/bin/launcher restart
```
# 五、测试
### 1、CLI工具说明
这里测试Presto与Hive的集成使用Presto提供的Presto CLI，该CLI是一个可执行的JAR文件，在上文上有相关下载。
### 2、文件传输及重命名
将下载好的文件上传到`/opt/cloudera/parcels/presto/bin`目录下并修改文件名称

```bash
mv presto-cli-0.232-executable.jar presto
```
### 3、赋权

```bash
chmod +x presto
ll
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200824111908992.png#pic_left)


  4、执行如下命令访问Hive

```bash
./presto --server S3:10899 --catalog=hive --schema=databaseName
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200824112027925.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70#pic_left)
5、监控界面可以看到相关信息
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200824112146742.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70#pic_left)
# 六、与SpringBoot和Mybatis集成
请参照：
[https://github.com/gm19900510/springboot-1.x-bigdata](https://github.com/gm19900510/springboot-1.x-bigdata)
[https://github.com/gm19900510/springboot-2.x-bigdata](https://github.com/gm19900510/springboot-2.x-bigdata)
以上项目包含与`SpringBoot`和`Mybatis`集成和多数据源切换，欢迎`Clone`、`Star`
