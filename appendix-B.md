# 附录 B  
  
## 安装 Storm 集群  
 
译者注：本附录的内容已经有些陈旧了。最新的 Storm 已不再必须依赖 ZeroMQ，各种依赖的库和软件也已经有更新的版本。

有以下两种方式创建 Storm 集群：

- 使用 [Storm 部署](https://github.com/nathanmarz/storm-deploy)在亚马逊 EC2 上面创建一个集群，就像你在第6章看到的。
- 手工安装（详见本附录）  

要手工安装 Storm，需要先安装以下软件

- Zookeeper集群（安装方法详见[管理向导](http://zookeeper.apache.org/doc/r3.3.3/zookeeperAdmin.html)）
- Java6.0
- Python2.6.6
- Unzip命令    

**NOTE**: Nimbus 和管理进程将要依赖 Java、Python 和 unzip 命令

安装本地库：

安装 ZeroMQ：

```
wget http://download.zeromq.org/historic/zeromq-2.1.7.tar.gz
tar -xzf zeromq-2.1.7.tar.gz
cd zeromq-2.1.7
./configure
make
sudo make install
```  

安装 JZMQ：

```
     git clone https://github.com/nathanmarz/jzmq.git
     cd jzmq
     ./autogen.sh
     ./configure
     make
     sudo make install
```  

本地库安装完了，下载最新的 Storm 稳定版（写作本书时是 Storm0.7.1。译者注：翻译本章时已是 v0.9.1，可从 [http://storm.incubator.apache.org/或https://github.com/apache/incubator-storm/releases](http://storm.incubator.apache.org/或https://github.com/apache/incubator-storm/releases)下载），并解压缩。  

编辑配置文件，增加 Storm 集群配置（可以从 Storm 仓库的 defaults.yaml 看到所有的默认配置）。  

编辑 Storm 目录下的 conf/storm.yaml，添加以下参数，增加集群配置：

storm.zookeeper.servers:  
– "zookeeper addres 1"  
– "zookeeper addres 2"  
– "zookeeper addres N"  
storm.local.dir: "a local directory"  
nimbus.host: "Nimbus host addres"  
supervisor.slots.ports:  
– supervisor slot port 1  
– supervisor slot port 2  
– supervisor slot port N  

参数解释：  

storm.zookeeper.servers

你的 zookeeper 服务器地址。
  
storm.local.dir：

    Storm 进程保存内部数据的本地目录。（务必保证运行 Storm 进程的用户拥有这个目录的写权限。）  
   
nimbus.host

    Nimbus运行的机器的地址  

supervisor.slots.ports

    接收消息的工人进程监听的端口号（通常从6700开始）；管理进程为这个属性指定的每个端口号运行一个工人进程。

当你完成了这些配置，就可以运行所有的 Storm 进程了。如果你想运行一个本地进程测试一下，就把 nimbus.host 配置成 localhost。
启动一个 Storm 进程，在 Storm 目录下执行：./bin/storm 进程名。

**NOTE**：Storm 提供了一个出色的叫做 Storm UI 的工具，用来辅助监控拓扑。  
