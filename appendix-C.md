# 附录 C  
   
## 安装实际的例子  

译者注：有些软件的最新版本已有变化，译文不会完全按照原文翻译，而是列出当前最新版本的软件。

首先，从下述 GitHub 的 URL 克隆这个例子：

```
1
> git clone git://github.com/storm-book/examples-ch06-real-life-app.git  
```  

src/main

包含拓扑的源码

src/test

包含拓扑的测试用例

webapps 目录

包含 Node.js Web 可以执行拓扑应用

.
├── pom.xml
├── src
│ ├── main
│ │ └── java
│ └── test
│ └── groovy
└── webapp

### 安装 Redis  

Redis 的安装是相当简单的：



1. 从 [Redis 站点](http://redis.io/download)下载最新的稳定版（译者注：翻译本章时最新版本是2.8.9。）
2. 解压缩
3. 运行 **make**，和 **make install**。  

上述命令会编译 Redis 并在 PATH 目录（译者注：/usr/local/bin）创建可执行文件。

可以从 Redis 网站上获取更多信息，包括相关命令文档及设计理念。

### 安装 Node.js  

安装 Node.js 也很简单。从 [http://www.nodejs.org/#download](http://www.nodejs.org/#download) 下载最新版本的 Node.js 源码。

当前最新版本是v0.10.28

下载完成，解压缩，执行
  
```
1
<b>./configure</b>
2
 
3
<b>make</b>
4
 
5
<b>make install</b>  
```  

可以从官方站点得到更多信息，包括在不同平台上安装 Node.js 的方法。

### 构建与测试  

为了构建这个例子，需要先启动 redis-server

>nohup redis-server &

然后执行 mvn 命令编译并测试这个应用。

>mvn package  

…
[INFO] ————————————————————————
[INFO] BUILD SUCCESS
[INFO] ————————————————————————
[INFO] Total time: 32.163s
[INFO] Finished at: Sun Jun 17 18:55:10 GMT-03:00 2012
[INFO] Final Memory: 9M/81M
[INFO]

### 运行拓扑   

启动了 redis-service 并成功构建之后，在 LocalCluster 启动拓扑。

>java -jar target/storm-analytics-0.0.1-jar-with-dependencies.jar

启动拓扑之后，用以下命令启动 Node.js Web 应用：

>node webapp/app.js

**NOTE**：拓扑和 Node.js 命令会互相阻塞。尝试在不同的终端运行它们。

### 演示这个例子  

在浏览器输入 http://localhost:3000/ 开始演示这个例子！

### 关于作者  

Jonathan Leibiusky，MercadoLibre 的主要研究与开发人员，已在软件开发领域工作逾10年之久。他已为诸多开源项目贡献过源码，包括 “Jedis”，它在 VMware 和 SpringSource 得到广泛使用。

Gabriel Eisbruch 一位计算机科学学生，从2007年开始在 Mercadolibre(NASDAQ MELI) 任架构师。主要负责研究与开发软件项目。去年，他专门负责大数据分析，为 MercadoLibre 实现了 Hadoop 集群。

Dario Simonassi 在软件开发领域有10年以上工作经验。从2004年开，他专门负责大型站点的操作与性能。现在他是 MercadoLibre(NASDAQ MELI) 的首席架构师，领导着该公司的架构师团队。