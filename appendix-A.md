# 附录 A  
  
## 安装 Storm 客户端  

Storm 客户端能让我们使用命令管理集群中的拓扑。按照以下步骤安装 Storm 客户端：



1. 从 Storm 站点下载最新的稳定版本（[https://github.com/nathanmarz/storm/downloads](https://github.com/nathanmarz/storm/downloads)）当前最新版本是storm-0.8.1。（译者注：原文是 storm-0.6.2，不过翻译的时候已经是 storm-0.8.1 了）  

2. 把下载的文件解压缩到 /usr/local/bin/storm 的 Storm 共享目录。  

3. 把 Storm 目录加入 PATH 环境变量，这样就不用每次都输入全路径执行 Storm 了。如果我们使用了 /usr/local/bin/storm，执行 export PATH=$PATH:/usr/local/bin/storm。  

4. 最后，创建 Storm 本地配置文件：~/.storm/storm.yaml，在配置文件中按如下格式加入nimbus 主机：  

      nimbus.host:"我们的nimbus主机"

现在，你可以管理你的 Storm 集群中的拓扑了。

**NOTE**：Storm 客户端包含运行一个 Storm 集群所需的所有 Storm 命令，但是要运行它你需要安装一些其它的工具并做一些配置。详见附录B。

有许多简单且有用的命令可以用来管理拓扑，它们可以提交、杀死、禁用、再平衡拓扑。

**jar** 命令负责把拓扑提交到集群，并执行它，通过 **StormSubmitter** 执行主类。
  
```
1
storm jar path-to-topology-jar class-with-the-main arg1 arg2 argN   
```   

path-to-topology-jar 是拓扑 jar 文件的全路径，它包含拓扑代码和依赖的库。 class-with-the-main 是包含 main 方法的类，这个类将由 StormSubmitter 执行，其余的参数作为 main 方法的参数。

我们能够挂起或停用运行中的拓扑。当停用拓扑时，所有已分发的元组都会得到处理，但是spouts 的 **nextTuple** 方法不会被调用。

停用拓扑：
  
```
1
storm deactivte topology-name  
```  

启动一个停用的拓扑：
  
```
1
storm activate topology-name  
```  

销毁一个拓扑，可以使用 kill 命令。它会以一种安全的方式销毁一个拓扑，首先停用拓扑，在等待拓扑消息的时间段内允许拓扑完成当前的数据流。
杀死一个拓扑：
  
```
1
storm kill topology-name  
```  

**NOTE**:执行 kill 命令时可以通过 -w [等待秒数]指定拓扑停用以后的等待时间。

再平衡使你重分配集群任务。这是个很强大的命令。比如，你向一个运行中的集群增加了节点。再平衡命令将会停用拓扑，然后在相应超时时间之后重分配工人，并重启拓扑。
再平衡拓扑：
  
```
1
storm rebalance topology-name  
```  

**NOTE**:执行不带参数的 Storm 客户端可以列出所有的 Storm 命令。完整的命令描述请见：[https://github.com/nathanmarz/storm/wiki/Command-line-client](https://github.com/nathanmarz/storm/wiki/Command-line-client)。
