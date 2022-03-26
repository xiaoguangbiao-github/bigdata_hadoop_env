# Hadoop集群搭建
## 1、Hadoop集群简介
## 2、Hadoop部署方式
## 3、Hadoop源码编译
## 4、Hadoop集群安装
### 4.1、集群角色规划
* 角色规划的准则  
根据软件工作特性和服务器硬件资源情况合理分配
比如依赖内存工作的NameNode是不是部署在大内存机器上？
* 角色规划注意事项  
资源上有抢夺冲突的，尽量不要部署在一起
工作上需要互相配合的。尽量部署在一起。

120.79.205.240   172.16.206.198   node1  namenode datanode resourcemanager nodemanager  
119.23.190.22    172.16.206.205   node2  secondarynamenode datanode nodemanager  
39.108.249.253   172.16.206.204   node3  datanode nodemanager  

### 4.2、服务器基础环境准备
（1）设置主机名（3台都做）  
vim /etc/hostname  #分别设置node1、node2、node3

（2）配置Hosts映射（3台都做）  
vim /etc/hosts  
172.16.206.198 node1  
172.16.206.205 node2  
172.16.206.204 node3  

（3）关闭防火墙（3台都做）    
systemctl stop firewalld.service   #关闭防火墙   
systemctl disable firewalld.service #禁止防火墙开启自启   

（4）ssh免密登录（node1执行->node1|node2|node3）  
ssh-keygen #4个回车 生成公钥、私钥  
ssh-copy-id node1、ssh-copy-id node2、ssh-copy-id node3   
如果报错：The ECDSA host key for XXX has changed......  
解决：将/home/${username}/.ssh/known_hosts中master的公钥删除掉，重新ssh-copy-id node1。  

（5）集群时间同步（3台都做）  
yum -y install ntpdate   
ntpdate ntp4.aliyun.com   

（6）jdk环境准备（3台都做）  
https://www.cnblogs.com/ljxt/p/11612636.html  

（7）创建统一目录（3台都做） 
mkdir -p /export/server/    #软件安装路径  
mkdir -p /export/data/      #数据存储路径  
mkdir -p /export/software/  #安装包存放路径  

（8）上传hadoop压缩包并解压（node1做）  
hadoop-3.1.4-bin-snappy-CentOS7.tar.gz  
tar zxvf hadoop-3.1.4-bin-snappy-CentOS7.tar.gz -C /export/server/  

（9）编辑Hadoop配置文件（node1做） 
cd /export/server/hadoop-3.1.4/etc/hadoop/  
vim ......

````java
<!------------------------hadoop-env.sh------------------------------->
#配置JAVA_HOME
export JAVA_HOME=/export/server/jdk1.8.0_65 

#设置用户以执行对应角色shell命令
export HDFS_NAMENODE_USER=root  
export HDFS_DATANODE_USER=root
export HDFS_SECONDARYNAMENODE_USER=root
export YARN_RESOURCEMANAGER_USER=root
export YARN_NODEMANAGER_USER=root 

<!------------------------core-site.xml------------------------------->

<!-- 默认文件系统的名称。通过URI中schema区分不同文件系统。-->
<!-- file:///本地文件系统 hdfs:// hadoop分布式文件系统 gfs://。-->
<!-- hdfs文件系统访问地址：http://nn_host:8020。-->
<property>
    <name>fs.defaultFS</name>
    <value>hdfs://node1.itcast.cn:8020</value>
</property>
<!-- hadoop本地数据存储目录 format时自动生成 -->
<property>
    <name>hadoop.tmp.dir</name>
    <value>/export/data/hadoop-3.1.4</value>
</property>
<!-- 在Web UI访问HDFS使用的用户名。-->
<property>
    <name>hadoop.http.staticuser.user</name>
    <value>root</value>
</property>

<!------------------------hdfs-site.xml------------------------------->

<!-- 设定SNN运行主机和端口。-->
<property>
    <name>dfs.namenode.secondary.http-address</name>
    <value>node2.itcast.cn:9868</value>
</property>

<!------------------------mapred-site.xml------------------------------->

<!-- mr程序默认运行方式。yarn集群模式 local本地模式-->
<property>
  <name>mapreduce.framework.name</name>
  <value>yarn</value>
</property>
<!-- MR App Master环境变量。-->
<property>
  <name>yarn.app.mapreduce.am.env</name>
  <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
</property>
<!-- MR MapTask环境变量。-->
<property>
  <name>mapreduce.map.env</name>
  <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
</property>
<!-- MR ReduceTask环境变量。-->
<property>
  <name>mapreduce.reduce.env</name>
  <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
</property>

<!------------------------yarn-site.xml------------------------------->

<!-- yarn集群主角色RM运行机器。-->
<property>
    <name>yarn.resourcemanager.hostname</name>
    <value>node1.itcast.cn</value>
</property>
<!-- NodeManager上运行的附属服务。需配置成mapreduce_shuffle,才可运行MR程序。-->
<property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
</property>
<!-- 每个容器请求的最小内存资源（以MB为单位）。-->
<property>
  <name>yarn.scheduler.minimum-allocation-mb</name>
  <value>512</value>
</property>
<!-- 每个容器请求的最大内存资源（以MB为单位）。-->
<property>
  <name>yarn.scheduler.maximum-allocation-mb</name>
  <value>2048</value>
</property>
<!-- 容器虚拟内存与物理内存之间的比率。-->
<property>
  <name>yarn.nodemanager.vmem-pmem-ratio</name>
  <value>4</value>
</property>
````

（10）编辑Hadoop配置文件（node1做） 
cd /export/server/hadoop-3.1.4/etc/hadoop/  
vim workers  
````java
node1
node2
node3
````

（11）分发同步安装包（node1做） 
在node1机器上将Hadoop安装包scp同步到其他机器  
cd /export/server/  
scp -r hadoop-3.1.4 root@node2:/export/server/  
scp -r hadoop-3.1.4 root@node3:/export/server/  

（12）配置Hadoop环境变量（node1做） 
vim /etc/profile
export HADOOP_HOME=/export/server/hadoop-3.1.4
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin

将修改后的环境变量同步其他机器（node1做） 
scp /etc/profile root@node2:/etc/
scp /etc/profile root@node3:/etc/

重新加载环境变量 验证是否生效（3台都做）	
source /etc/profile
hadoop #验证环境变量是否生效

（13）NameNode format格式化操作（node1做） 
首次启动HDFS时，必须对其进行格式化操作。
format本质上是初始化工作，进行HDFS清理和准备工作  
命令：hdfs namenode -format

首次启动之前需要format操作，format只能进行一次 后续不再需要，如果多次format除了造成数据丢失外，还会导致hdfs集群主从角色之间互不识别，通过删除所有机器hadoop.tmp.dir目录重新forma解决。  

（14）每台机器上每次手动启动关闭一个角色进程

HDFS集群
hdfs --daemon start namenode|datanode|secondarynamenode
hdfs --daemon stop  namenode|datanode|secondarynamenode

YARN集群
yarn --daemon start resourcemanager|nodemanager
yarn --daemon stop  resourcemanager|nodemanager

（14）node1上，使用软件自带的shell脚本一键启动
前提：配置好机器之间的SSH免密登录和workers文件。
* HDFS集群
start-dfs.sh 
stop-dfs.sh 
* YARN集群
start-yarn.sh
stop-yarn.sh
* Hadoop集群
start-all.sh
stop-all.sh 



