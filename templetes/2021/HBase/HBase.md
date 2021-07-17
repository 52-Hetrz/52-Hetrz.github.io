---
typora-root-url: ..\..\..
typora-copy-images-to: ../../../images/HBase
---

## HBase

### 1. 逻辑视图

![image-20210716103157946](images/HBase/image-20210716103157946.png)

​	**Table 在行的方向上分割为多个HRegion，一个region由[startkey,endkey)表示，每个HRegion分散在不同的RegionServer中**

**Row（行）:**

* 每行数据有一个可排序的关键字和任意列项。
* 可以只对一行上“锁”。
* 对行的写操作始终是“原子”的。
* 表内数据非常“稀疏”，不同的行的列的数目完全可以大不相同。

**Row Key（行键）**: 

* 行键，Table的主键，Table中的记录按照Row Key排序。

* 字符串、整数、二进制串甚至串行化的结构都可以作为行键。
* 表按照行键的“逐字节排序”顺序对行进行有序化处理。

**Time Stamp（时间戳）：**

* 每次数据操作对应的时间戳，可以看作是数据的Version number
* 数据可以通过时间戳区分版本。

**Column（列）：**

* 列必须使用“族（family）”来定义。
* 任意一列有如下的形式：“族：标签”，（其中族和标签都可以为任意形式的串）
* 物理上将同“族”的数据存储在一起
* 表在水平方向上有一个或多个Column Family组成，一个Column Family中可以有任意多个Column组成，即Column Family支持动态的扩展，无需预先定义Column的数量以及类型，所有的Column均以二进制格式存储，用户需要自行进行类型转换。
* 族：是固定不变的，只能用过改变表结构来改变。
* 标签：可以任意加入，因此列是可以任意扩展的。

### 2. 系统总体结构

![image-20210716104644463](images/HBase//image-20210716104644463.png)



**Client（客户端）： **

* Client包含访问HBase的接口，Client维护者一些cache来加快对HBase的访问，比如Region的位置信息。
* client访问HBase上的数据过程并不需要master参与，寻址访问ZooKeeper和Region Service ,数据读写访问Region Service，Region Service主要负责想用用户IO请求，向HDFS文件系统中读写数据，是HBase中最核心的模块。

**Region（区域）：**

* 数据存储实体
* 表按照“水平”的方式，划分成一个或多个“区域”。
* 每个区域都包含一个随机id，区域内的行也是按照行键有序排列的。
* 最初每张表包含一个Region，当表增大超过阈值的时候，这个Region被自动分割成两个大小相同的Region。
* Region以分布式的方式分布在集群内。

**区域服务器（Region Service）：**

* 维护Master分配给它的Region
* 处理对这些Region的IO请求。
* 负责切分在运行过程中变得过大的region。

**Master:**

* 可以启动多个Master，但通过ZooKeeper的Master Election机制保证总有一个Master运行。
* 为Region Service分配Region。
* 负责Region Service的负载均衡。
* 发现失效的Region Service并重新为其分配Region。

**ZooKeeper:**

* 保证任何时候，集群中只有一个running master 
* 存贮所有Region 的寻址入口。
* 实时监控Region Server 的状态，将Region server 的上线和下线信息，实时通知给Master。
* 存储Hbase 的schema,包括有哪些table，每个table 有哪些column family。
* ZooKeeper中的三类角色：

![image-20210716110611608](images/HBase//image-20210716110611608.png)

* 系统模型图：

  ![image-20210716110202215](images/HBase//image-20210716110202215.png)

  ​	ZooKeeper设计的目的（最终一致性)：client不论连接到哪个Server，展示给它都是同一个视图，这是zookeeper最重要的性能。删除Server1中的数据后，其他集群的Server会自动同步删除之后的数据。因此Zookeeper可以保证多个Server的一致性。 HMaster没有单点问题，HBase中可以启动多个HMaster，通过Zookeeper的Master Election机制保证总有一个Master运行。

### 3.特殊目录结构

![image-20210716113830096](/images/HBase/image-20210716113830096.png)

**-ROOT-：**

* 记录了.META.表的Region信息，-ROOT-只有一个root region，Zookeeper中记录了-ROOT-表的location。
* 只存包含一个区域。
* 保存了.META.表其他region的位置。
* root region永远不会被分割，保证了最多需要三次跳转就能定位到任意region。

**.META.（元数据表）：**

* 记录了用户表的Region信息，.META.可以有多个regoin。
* 全部用户区域的属性数据都存在元数据表中。
* 包括区域中数据起止行信息、区域“在线”状态等。
* 保存区域服务器地址。
* META表每行保存一个region的位置信息。
* 为了加快访问，MATE表全部的region都保存在内存中。

​	Client访问用户数据之前需要首先访问zookeeper，然后访问-ROOT-表，接着访问.META.表，最后才能找到用户数据的位置去访问

### 4. Region Service

**写：**

* 数据首先写入“预写”日志。
* 对于一个Region Service而言，所有Region的“写”操作都存储在同一个日志当中。
* 数据并非直接写入文件系统，而是先缓存，缓存到一定数量在批量写入。
* 写入完成后在日志中做标记。

**读：**

* 区域服务器先在内存的缓存中查找，如果命中请求，则直接服务。
* 如果存在多个版本，则返回顺序按照从最新到最老。

**合并：**

* 如果映射文件(Map File)数量超过阈值，区域服务器会进行一次合并(Compaction)
* 合并操作也周期性进行
* 合并可与区域服务器响应用户的读写请求并发进行
* 如果读写请求与合并区域相关，读写操作先挂起，直到合并操作完成

**分割：**

* 当区域大过阈值后，区域会按照行的方式对半进行分割(Split)操作
* 分割也作为一种请求被区域服务器处理
* 被分割区域先离线
* 区域服务器在元信息表中生成子区域元信息
* 主服务器在得知分割操作进行后，将子区域分配给新的区域服务器进行服务
* 被分割区域通过垃圾回收机制回收
* 如果主服务器没能正确收到分割消息，主服务器可通过定期检查MATA数据发现分割操作
* 开始分割操作后，被分割区域离线，此时客户端能检测到并在分割后的区域上线后重发访问请求

**失效恢复：**

* 由于检测没有心跳，主服务器能够探知区域服务器的失效
* 主服务器将失效服务器所提供服务的区域重新分配给其它区域服务器
* 原失效区域服务器的“预写”日志由主服务器进行分割并派送给新的区域服务器

**客户端：**

* 连接到ZooKeeper集群获取根区域数据和元数据的位置。
* 在元数据中查找需要访问行所在的区域并定位提供该区域服务的区域服务器
* 直接与区域服务器交互以获取数据
* 根区域数据、元数据以及用户区域信息都被客户端缓存以备下次访问使用

### 5. HBase安装及环境配置

1. 在[HBase的清华镜像站](https://mirrors.tuna.tsinghua.edu.cn/apache/hbase/)下载对应的安装包，这里我选择使用hbase-1.6.0-bin.tar.gz

![1626423716647](/images/HBase/1626423716647.png)

2. 在/opt 文件目录下创建一个新的文件夹hbase，将压缩包移动到/opt/hbase目录下，然后解压到当前目录下：

```shell
$ cd /opt
$ sudo mkdir hbase
$ cd ~/下载
$ sudo mv hbase-1.6.0-bin.tar.gz /opt/hbase
$ sudo tar -zxvf hbase-1.6.0-bin.tar.gz
$ sudo rm hbase-1.6.0-bin.tar.gz
```

![1626424436327](/images/HBase/1626424436327.png)

3. 修改/etc/profile 文件

```shell
$ sudo gedit /etc/profile
```

![1626424583892](/images/HBase/1626424583892.png)

增加信息:"**export HBASE_HOME=/opt/hbase/hbase-1.6.0** "(注意，＝　左右不要有空格)

并在PATH尾部添加"**:$HBASE_HOME/bin**"

![1626428118678](/images/HBase/1626428118678.png)

保存退出，然后执行

```shell
$ source /etc/profile
```

4. 修改 hbase/config/hbase-env.sh文件，添加如下信息，然后保存退出：

![1626425192571](/images/HBase/1626425192571.png)

5. 修改config/hbase-site.xml文件，添加如下信息(hbase.rootdir中的主机端口号应该和hadoop的配置文件core-site.xml文件中fs.defaultFS的主机和端口号一致)：

```xml
	<property>
  		<name>hbase.rootdir</name>
	  	<value>hdfs://localhost:8020/hbase</value>
	</property>
	<property>
  		<name>hbase.cluster.distributed</name>
	  	<value>true</value>
	</property>
	<property>
		<name>hbase.zookeeper.quorum</name>
  		<value>hadoop0</value>
	</property>
	<property>
  		<name>dfs.replication</name>
  		<value>1</value>
	</property>
 	<property>
      		<name>hbase.tmp.dir</name>
      		<value>/opt/hbase/tmp</value>
   	</property>
```

6. 先启动hadoop，再启动hbase

```shell
$ sudo cd /opt/hadoop/hadoop-2.10.1
$ sudo ./sbin/start-all.sh
$ sudo cd opt/hbase/hbase-1.6.0
$ sudo ./bin/start-hbase.sh
```

* 在启动时报了如下的警告：

![1626426620984](/images/HBase/1626426620984.png)

这是因为[“JDK 8兼容性指南”](http://www.oracle.com/technetwork/java/javase/8-compatibility-guide-2156366.html)  指出，在Java 8中，命令行标志  `MaxPermSize` 已被删除。原因是永久代从热点堆中被移除并被转移到本地内存。所以为了删除这条消息，编辑 hbase-env.sh，注释掉如下两个语句：

![1626426701065](/images/HBase/1626426701065.png)

![1626426713512](/images/HBase/1626426713512.png)

* localhost拒绝连接

![1626427302226](/images/HBase/1626427302226.png)

编辑配置文件　/etc/ssh/sshd_config

找到：PermitRootLogin prohibit-password在前面添加#
 		添加：PermitRootLogin yes

![1626427455119](/images/HBase/1626427455119.png)

然后重启服务：

``$ sudo service ssh restart``

* 期间还遇到了许多奇葩的问题，可以打开logs目录下的日志，找到报错位置，然后对应的搜索解决办法。

7. 最后启动成功之后，输入jps仍然没有显示HMaster进程，但可以通过http://localhost:16010/master-status访问HBase界面。

![1626496113792](/images/HBase/1626496113792.png)

![1626496203886](/images/HBase/1626496203886.png)





