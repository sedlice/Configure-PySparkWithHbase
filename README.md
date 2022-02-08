# Configure-PySparkWithHbase
适用于2022年前后的，如何配置单机版的PySpark3.2和Hbase2.4，用于在本地练习操作大数据操作

| 运行环境组件 | 使用版本 |
| --- | --- |
| JDK | 8   |
| zookeeper | 3.6.3 |
| hadoop | 3.2.2 |
| spark | 3.2.0 |
| hbase | 2.4.8 |
| python | 3.9.8 |

<br>

# 步骤1：安装JDK8（如果只用于运行而不用于开发java程序的话，也可以就直接安装JRE8）

> 可以到 [IBM open9J版本openjdk下载地址](https://developer.ibm.com/languages/java/semeru-runtimes/downloads) 下载open9J版本，降低内存占用。也可以到 [HotSpot版本openjdk下载地址](https://www.openlogic.com/openjdk-downloads)下载HotSpot版本，都可以。

#### 最好是使用JDK8版本，目前官方对JDK8的支持是最好的，JDK7和JDK11可以用，但是可能存在兼容性问题。

1.  将jdk压缩包下载到本地，然后解压，移动到/usr/local/目录下
2.  编辑`/etc/profile`文件，添加JAVA\_HOME的环境变量，然后将JAVA\_HOME/bin添加到PATH中

<br>

# 步骤2：使用root帐号创建hadoop用户或添加root用户到运行文件中

### 方案一：创建hadoop用户

```shell
$ useradd -m hadoop -s /bin/bash
$ passwd hadoop
$ adduser hadoop sudo
```

然后切换到hadoop用户。

### 方案二：添加root用户到运行文件中（在hadoop的配置中说明，步骤5第2步）我自己为了方便，就采用的是这种方案。

<br>

# 步骤3：创建SSH免密登录

查看系统是否有openssh-server服务，如果没有就通过命令安装

```shell
$ sudo yum install -y openssl openssh-server

$ ssh-keygen -t rsa	# 会有提示，都按回车就可以
$ cd ~/.ssh/
$ cat ./id_rsa.pub >> ./authorized_keys	# 加入授权
```

<br>

# 步骤4：下载配置zookeeper

> 下载编译后的bin版本，不要下载Source版本，其他几个组件包也是一样下载bin版本

1.  下载zookeeper压缩包，并解压，同样移动到/usr/local/目录下，并改名为zookeeper
    使用`sudo chown -R hadoop ./zookeeper`命令进行文件夹权限转移
    [zookeeper下载地址](https://zookeeper.apache.org/releases.html)


2.  进入解压后的目录，拷贝文件zoo_sample.cfg并重命名为zoo.cfg。

```shell
cd ./conf
cp ./zoo_sample.cfg ./zoo.cfg
```

3.  编辑`zoo.cfg`，进行如下配置。

```shell
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/usr/local/zookeeper/datas #zookeeper数据文件的路径
clientPort=2181 #zookeeper访问端口默认是2181
```

4.  保存后，通过命令`/usr/local/zookeeper/bin/zkServer.sh start`来启动zookeeper服务。

<br>

# 步骤5：下载配置hadoop

1.  下载hadoop压缩包，并解压，同样移动到/usr/local/目录下，并改名为hadoop
    使用`sudo chown -R hadoop ./hadoop`命令进行文件夹权限转移
    [Hadoop下载地址](https://hadoop.apache.org/releases.html)

> 这里要注意，必须要下binary的编译版本，不要下src版本的源代码

2.  进入`/usr/local/hadoop`目录中，编辑`./etc/hadoop/hadoop-env.sh`文件

- 先添加root用户

> 如果是创建了hadoop，并使用hadoop用户操作的话，则该步骤省略

```
export HDFS_NAMENODE_USER=root
export HDFS_DATANODE_USER=root
export HDFS_SECONDARYNAMENODE_USER=root
#单机版和伪分布式这后面两行可以不写
export YARN_RESOURCEMANAGER_USER=root
export YARN_NODEMANAGER_USER=root
```

- 再在此文件中添加JAVA_HOME的绝对路径，保证hadoop的顺利运行（顺便设置下pid文件的存放位置）

```
export JAVA_HOME=/usr/local/jdk
export HADOOP_PID_DIR=/usr/local/hadoop/pids
```

3.  编辑`./etc/hadoop/core-site.xml`文件，修改为如下代码

```xml
<configuration>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>file:/usr/local/hadoop/tmp</value>
        <description>Abase for other temporary directories.</description>
    </property>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>
```

4.  编辑`./etc/hadoop/hdfs-site.xml`文件，修改为如下代码

```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:/usr/local/hadoop/tmp/dfs/name</value>
    </property>
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:/usr/local/hadoop/tmp/dfs/data</value>
    </property>
</configuration>
```

5.  使用`./bin/hdfs namenode -format`进行NameNode的格式化<span style="color: red;">（必须格式化才能正常使用）</span>

6.  最后使用命令`./sbin/start-dfs.sh`启动hadoop。访问`http://(ip):9870`，如果能看到Hadoop页面，表示Hadoop单机配置完成。如果要停止的话，就使用命令`./sbin/stop-dfs.sh`。
> 需要注意一个点，就是打开/etc/hosts文件，查看配置中的ip地址是不是都是本机的地址，而且localhost对应的地址只能有一个，如果ip6的地址中也存在localhost名称的地址，需要改成ip6-localhost。
    
<br>

# 步骤6：下载配置spark

> 我这里下载的是<span style="color: red;">spark-3.2.0-bin-hadoop3.2.tgz</span>包。
1.  下载spark压缩包，并解压，同样移动到/usr/local/目录下，并改名为spark
    使用`sudo chown -R hadoop ./spark`命令进行文件夹权限转移
    [Spark下载地址](https://spark.apache.org/downloads.html)
2.  执行启动命令`./sbin/start-master.sh`，访问`http://(ip):8080`，可以看到spark页面的话，表明spark启动成功。如果显示404，那很可能是因为8080端口无法绑定，顺延到后面的端口了，如8081、8082之类的，可以查看启动日志来确定。
    
3.  尝试执行命令`./bin/pysaprk`，如果能够成功启动pyspark，说明spark安装配置成功。
    
<br>

# 步骤7：安装jupyter-lab并配置

> 假定本机当前已经安装好python3

1.  使用命令`pip3 install jupyterlab`安装jupyter-lab。
2.  通过命令生成jupyter-lab的配置文件。

```
/usr/local/python39/bin/jupyter-lab --generate-config
```

3.  修改刚生成的jupyter-lab配置文件，使jupyter-lab允许外部访问。

```
vim /root/.jupyter/jupyter_lab_config.py
```

找到如下两行配置，取消注释并修改为

```
c.ServerApp.allow_origin = '*'
c.ServerApp.ip = '0.0.0.0'
```

4.  使用命令添加访问密码

```
/usr/local/python39/bin/jupyter-lab password
# 然后输入两次密码
```

5.  修改pyspark启动文件，使其调用jupyter-lab

```
vim /usr/local/spark/bin/pyspark

export PYSPARK_DRIVER_PYTHON=/usr/local/python39/bin/jupyter
export PYSPARK_DRIVER_PYTHON_OPTS='lab --allow-root'
# --allow-root 是以root用户启动，如果是非root用户的话，可以不需要加这个配置，只写'lab'即可
```

6.  之后启动pyspark，默认会启用jupyter-lab作为调用，而且不再是使用默认的token登录，而是使用设置的密码进行登录。

```shell
$ cd /usr/local/spark
$ ./bin/pyspark
```

7. 测试一下在jupyter中pyspark是否能正常工作
```python
# 这里的pyspark会自动创建一个sc（SparkContext）对象，我们直接拿来调用就好了
words = sc.parallelize (
   ["scala", 
   "java", 
   "hadoop", 
   "spark", 
   "akka",
   "spark vs hadoop", 
   "pyspark",
   "pyspark and spark"]
)
counts = words.count()
print(f"Number of elements in RDD -> {counts}")
```

<br>

# 步骤8：Hbase的下载和配置（如果需要的话）

1.  下载hbase压缩包，并解压，同样移动到/usr/local/目录下，并改名为hbase
    使用`sudo chown -R hadoop ./hbase`命令进行文件夹权限转移
    [Hbase下载地址](https://hbase.apache.org/downloads.html)

> 注意下载编译（bin）版本

2.  进入`/usr/local/hbase`目录，修改`./conf/hbase-env.sh`文件，进行如下的配置。
```
export JAVA_HOME=/usr/local/jdk/
export HBASE_PID_DIR=/usr/local/hbase/pids
# 不使用内置的zookeeper
export HBASE_MANAGES_ZK=false
# 让hbase的程序不要去hadoop的目录下查找jar包（因为存在不同版本的jar包，会有冲突错误，导致无法正常使用）
export HBASE_DISABLE_HADOOP_CLASSPATH_LOOKUP=true
```
3.  修改`./conf/hbase-site.xml`文件，将下方的配置修改为如下代码

```xml
<configuration>
  <property>
	 <!-- 使用分布式方式部署 -->
    <name>hbase.cluster.distributed</name>
    <value>true</value>
  </property>
  <property>
    <name>hbase.rootdir</name>
    <value>hdfs://localhost:9000/hbase</value>
  </property>
  <property>
	 <!-- 设置zookeeper的访问地址 -->
    <name>hbase.zookeeper.quorum</name>
    <value>localhost</value>
  </property>
  <property>
	 <!-- 设置zookeeper的访问端口 -->
    <name>hbase.zookeeper.property.clientPort</name>
    <value>2181</value>
  </property>
  <property>
	 <!-- 设置zookeeper数据文件的路径 -->
    <name>hbase.zookeeper.property.dataDir</name>
    <value>/usr/local/zookeeper/datas</value>
   </property>
</configuration>
```

4.  执行启动命令

```
cd /usr/local/hbase
./bin/start-hbase.sh
```

5.  显示Hbase启动完成后，并且能够访问`http://(ip):16010`，表示hbase已配置成功。
6.  我们调用hbase shell，创建表并且插入两条数据看看运行是否正常。
```
# 进入hbase shell
./bin/hbase shell

# 创建student表，并设定列族info
create "student","info"

# 插入行键为001的张三的名字和年龄
put "student","001","info:name","ZhangSan"
put "student","001","info:age","15"

# 插入行键为002的李四的名字和年龄
put "student","002","info:name","LiSi"
put "student","002","info:age","14"
```
如果以上命令都可以执行成功，说明hbase已可以正常运行。

# 步骤9：Apache Hbase Thrift的启动和调用
> Thrift是hbase中一个对外的调用接口，由C++编写。配合python中的happybase包，可以直接对hbase进行操作。

1. 通过pip3安装happybase包（包内部已集成thrift的库文件，无需再去使用thrift做编译操作）
```
pip3 install happybase
```

2. 进入hbase的bin目录，启动thrift接口服务。这里最好用守护进程的方式启动，不然后面无法在当前终端中去启动jupyter-lab的pyspark了。
```
cd /usr/local/hbase
./bin/hbase-daemon.sh start thrift
```

3. 启动成功的话，thrift服务就会绑定到默认的9090端口，之后，只需要在python程序中使用happybase连接即可操作hbase。
```python
import happybase
connection = happybase.Connection("localhost", port=9090)
table = connection.table("student")
for key, data in table.scan():
    print(key, ":", data)
```