---
typora-copy-images-to: ../../../../images/HBase/HBase_usage
typora-root-url: ../../../..
---

## HBase使用

HBase提供了命令行的功能，hbase shell，位于bin目录下，可以通过命令行指令来启动shell

```shell
$ cd $HBASE_HOME
$ ./bin/hbase shell
```

![1626514543971](/images/HBase/HBase_usage/1626514543971.png)

### 1. 基本语法一览

| 名称                          | 命令表达式                                                   |
| ----------------------------- | ------------------------------------------------------------ |
| 查看hbase状态                 | status                                                       |
| 创建namespace                 | create_namespace '命名空间名称'                              |
| 删除namespace                 | drop_namespace '命名空间名称'                                |
| 查看namespace                 | describe_namespace '命名空间名称'                            |
| 列出所有namespace             | list_namespace                                               |
| 在namespace下创建表           | create '命名空间名称:表名', '列族名1'                        |
| 查看namespace下的表           | list_namespace_tables '命名空间名称'                         |
| 创建表，默认命名空间为default | create '表名','列族名1','列族名2','列族名N'                  |
| 查看所有表                    | list                                                         |
| 描述表                        | describe '表名'                                              |
| 判断表存在                    | exists '表名'                                                |
| 判断是否禁用启用表            | is_enabled '表名' is_disabled '表名'                         |
| 添加记录                      | put '表名','rowkey','列族：列'，'值'                         |
| 查看记录rowkey下的所有数据    | get '表名','rowkey'                                          |
| 查看所有记录                  | scan '表名'                                                  |
| 查看表中的记录总数            | count '表名'                                                 |
| 获取某个列族                  | get  '表名','rowkey','列族：列'                              |
| 获取某个列族的某个列          | get '表名','rowkey','列族：列'                               |
| 计算表的行数量                | count '表名'1626514735704                                    |
| 删除记录                      | delete '表名','行名','列族：列'                              |
| 删除整行                      | deleteall '表名','rowkey'                                    |
| 删除一张表                    | 先要屏蔽该表，才能对该表进行删除 第一步 disable '表名'，第二步 drop '表名' |
| 清空表                        | truncate '表名'                                              |
| 查看某个表某个列中所有数据    | scan '表名',{COLUMNS=>'列族名：列名'}                        |
| 更新记录                      | 就是重新一遍，进行覆盖，hbase没有修改，都是追加              |

#### 查看状态

```shell
hbase(main):001:0>  status 
```

![1626514735704](/images/HBase/HBase_usage/1626514735704.png)

#### name_space（命名空间）

```shell
hbase(main):002:0> create_namespace 'test'						//创建命名空间 'test'
hbase(main):003:0> list_namespace										//列出所有的命名空间
hbase(main):004:0> describe_namespace 'test'				//查看命名空间'test'的描述
hbase(main):006:0> list_namespace_tables 'default'		//查看命名空间'default'下的所有表
hbase(main):007:0> drop_namespace 'test'						//删除命名空间'test'
hbase(main):008:0> list_namespace										//查看命名空间
```

![1626515274123](/images/HBase/HBase_usage/1626515274123.png)

#### Table（表）

在创建表时，如果指定命名空间，则会将表创建在该命名空间下：

```shell
hbase(main):002:0> create_namespace 'test'
hbase(main):003:0> create 'test:test_table','test_row_key','test_column1','test_column2'
hbase(main):004:0> list_namespace_tables 'test'
hbase(main):005:0> desc 'test:test_table'
```

![1626515743645](/images/HBase/HBase_usage/1626515743645.png)

如果不指定命名空间，会将该表默认创建在default命名空间下：

```shell
hbase(main):005:0> create 'students','id','address','info'
hbase(main):006:0> exists 'students'
hbase(main):007:0> list
```

#### 记录

向表中插入数据

```shell
hbase(main):008:0> put 'students','1','address:province','shandong'
hbase(main):009:0> put 'students','1','address:country','weifang'
hbase(main):010:0> put 'students','2','address:province','hunan'
hbase(main):011:0> put 'students','2','info:age','20'
hbase(main):012:0> put 'students','2','info:birthday','2001-1-26'
```

![1626516598910](/images/HBase/HBase_usage/1626516598910.png)

查看数据：

```shell
hbase(main):001:0> get 'students','1'
hbase(main):002:0> get 'students','2'
hbase(main):003:0> scan 'students'
```

![1626516700042](/images/HBase/HBase_usage/1626516700042.png)

删除记录

```shell
hbase(main):004:0> delete 'students','1','address:country'
hbase(main):005:0> get 'students','1'
```

![1626516823674](/images/HBase/HBase_usage/1626516823674.png)

删除行

```shell
hbase(main):006:0> deleteall 'students','1'
hbase(main):007:0> scan 'students'
```

![1626516896308](/images/HBase/HBase_usage/1626516896308.png)

#### 其他

```shell
hbase(main):011:0> count 'students'
hbase(main):012:0> truncate 'students'
hbase(main):014:0> scan 'students'
hbase(main):015:0> disable 'students'
hbase(main):016:0> drop 'students'
```

![1626517775971](/images/HBase/HBase_usage/1626517775971.png)

![1626517790473](/images/HBase/HBase_usage/1626517790473.png)

