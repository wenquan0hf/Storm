# 使用非 JVM 语言开发  
  
有时候你可能想使用不是基于 JVM 的语言开发一个 Storm 工程，你可能更喜欢使用别的语言或者想使用用某种语言编写的库。

Storm 是用 Java 实现的，你看到的所有这本书中的 spout 和 bolt 都是用 java 编写的。那么有可能使用像 Python、Ruby、 或者 JavaScript 这样的语言编写 spout 和 bolt 吗？答案是当然

可以！可以使用多语言协议达到这一目的。

多语言协议是 Storm 实现的一种特殊的协议，它使用标准输入输出作为 spout 和 bolt 进程间的通讯通道。消息以 JSON 格式或纯文本格式在通道中传递。

我们看一个用非 JVM 语言开发 spout 和 bolt 的简单例子。在这个例子中有一个 spout 产生从1到10,000的数字，一个 bolt 过滤素数，二者都用 PHP 实现。

**NOTE**: 在这个例子中，我们使用一个很笨的办法验证素数。有更好当然也更复杂的方法，它们已经超出了这个例子的范围。

有一个专门为 Storm 实现的 PHP DSL (译者注：领域特定语言)，我们将会在例子中展示我们的实现。首先定义拓扑。
  
```
1
...
2
TopologyBuilder builder = new TopologyBuilder();
3
builder.setSpout("numbers-generator", new NumberGeneratorSpout(1, 10000));
4
builder.setBolt("prime-numbers-filter", new
5
PrimeNumbersFilterBolt()).shuffleGrouping("numbers-generator");
6
StormTopology topology = builder.createTopology();
7
...  
```  

**NOTE**：有一种使用非 JVM 语言定义拓扑的方式。既然 Storm 拓扑是 Thrift 架构，而且Nimbus 是一个 Thrift 守护进程，你就可以使用任何你想用的语言创建并提交拓扑。但是这已经超出了本书的范畴了。

这里没什么新鲜了。我们看一下 **NumbersGeneratorSpout** 的实现。
  
```
01
public class NumberGeneratorSpout extends ShellSpout implements IRichSpout {
02
    public NumberGeneratorSpout(Integer from, Integer to) {
03
       super("php", "-f", "NumberGeneratorSpout.php", from.toString(), to.toString());
04
    }
05
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
06
        declarer.declare(new Fields("number"));
07
    }
08
    public Map<String, Object> getComponentConfiguration() {
09
        return null;
10
    }
11
}  
```  

你可能已经注意到了，这个 spout 继承了 **ShellSpout**。 这是个由 Storm 提供的特殊的类，用来帮助你运行并控制用其它语言编写的 spout。 在这种情况下它告诉 Storm 如何执行你的PHP 脚本。

NumberGeneratorSpout 的 PHP 脚本向标准输出分发元组，并从标准输入读取确认或失败信号。

在开始实现 NumberGeneratorSpout.php 脚本之前，多观察一下多语言协议是如何工作的。

spout 按照传递给构造器的参数从 **from** 到 **to** 顺序生成数字。

接下来看看 **PrimeNumbersFilterBolt**。 这个类实现了之前提到的壳。它告诉 Storm 如何执行你的PHP脚本。 Storm 为这一目的提供了一个特殊的叫做 **ShellBolt** 的类，你惟一要做的事就是指出如何运行脚本以及声明要分发的属性。
  
```
1
public class PrimeNumbersFilterBolt extends ShellBolt implements IRichBolt {
2
    public PrimeNumbersFilterBolt() {
3
        super("php", "-f", "PrimeNumbersFilterBolt.php");
4
    }
5
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
6
        declarer.declare(new Fields("number"));
7
    }
8
}  
```  

在这个构造器中只是告诉 Storm 如何运行PHP脚本。它与下列命令等价。
  
```
1
php -f PrimeNumbersFilterBolt.php  
```  

PrimeNumbersFilterBolt.php 脚本从标准输入读取元组，处理它们，然后向标准输出分发、确认或失败。在开始这个脚本之前，我们先多了解一些多语言协议的工作方式。

1. 发起一次握手
2. 开始循环
3. 读/写元组  

**NOTE**：有一种特殊的方式可以使用 Storm 的内建日志机制在你的脚本中记录日志，所以你不需要自己实现日志系统。

下面我们来看一看上述每一步的细节，以及如何用 PHP 实现它。

## 发起握手

为了控制整个流程（开始以及结束它），Storm 需要知道它执行的脚本进程号（PID）。根据多语言协议，你的进程开始时发生的第一件事就是 Storm 要向标准输入（译者注：根据上下文理解，本章提到的标准输入输出都是从非 JVM 语言的角度理解的，这里提到的标准输入也就是 PHP 的标准输入）发送一段 JSON 数据，它包含 Storm 配置、拓扑上下文和一个进程号目录。它看起来就像下面的样子：
  
```
{
    "conf": {
        "topology.message.timeout.secs": 3,
        // etc
    },
    "context": {
        "task->component": {
            "1": "example-spout",
            "2": "__acker",
            "3": "example-bolt"
        },
        "taskid": 3
    },
    "pidDir": "..."
}  
```  

脚本进程必须在 **pidDir** 指定的目录下以自己的进程号为名字创建一个文件，并以 JSON 格式把进程号写到标准输出。

{"pid": 1234}
举个例子，如果你收到 **/tmp/example\n** 而你的脚本进程号是123，你应该创建一个名为 **/tmp/example/123** 的空文件并向标准输出打印文本行 **{“pid”: 123}\n**（译者注：此处原文只有一个 n，译者猜测应是排版错误）和 **end\n**。 这样 Storm 就能持续追踪进程号并在它关闭时杀死脚本进程。下面是 PHP 实现：
  
```
1
$config = json_decode(read_msg(), true);
2
$heartbeatdir = $config['pidDir'];
3
$pid = getmypid();
4
fclose(fopen("$heartbeatdir/$pid", "w"));
5
storm_send(["pid"=>$pid]);
6
flush();  
```  

你已经实现了一个叫做 **read_msg** 的函数，用来处理从标准输入读取的消息。按照多语言协议的声明，消息可以是单行或多行 JSON 文本。一条消息以 **end\n** 结束。
  
```
01
function read_msg() {
02
    $msg = "";
03
    while(true) {
04
        $l = fgets(STDIN);
05
        $line = substr($l,0,-1);
06
        if($line=="end") {
07
            break;
08
        }
09
        $msg = "$msg$line\n";
10
    }
11
    return substr($msg, 0, -1);
12
}
13
function storm_send($json) {
14
    write_line(json_encode($json));
15
    write_line("end");
16
}
17
function write_line($line) {
18
    echo("$line\n");
19
}  
```  

**NOTE**：flush() 方法非常重要；有可能字符缓冲只有在积累到一定程度时才会清空。这意味着你的脚本可能会为了等待一个来自 Storm 的输入而永远挂起，而 Storm 却在等待来自你的脚本的输出。因此当你的脚本有内容输出时立即清空缓冲是很重要的。

## 开始循环以及读/写元组

这是整个工作中最重要的一步。这一步的实现取决于你开发的 spout 和 bolt。

如果是 spout，你应当开始分发元组。如果是 bolt，就循环读取元组，处理它们，分发它发，确认成功或失败。

下面我们就看看用来分发数字的 spout。
  
```
01
$from = intval($argv[1]);
02
$to = intval($argv[2]);
03
while(true) {
04
    $msg = read_msg();
05
    $cmd = json_decode($msg, true);
06
    if ($cmd['command']=='next') {
07
        if ($from<$to) {
08
            storm_emit(array("$from"));
09
            $task_ids = read_msg();
10
            $from++;
11
        } else {
12
            sleep(1);
13
        }
14
    }
15
    storm_sync();
16
}  
```  

从命令行获取参数 **from** 和 **to**，并开始迭代。每次从 Storm 得到一条 **next** 消息，这意味着你已准备好分发下一个元组。

一旦你发送了所有的数字，而且没有更多元组可发了，就休眠一段时间。

为了确保脚本已准备好发送下一个元组，Storm 会在发送下一条之前等待 **sync\n** 文本行。调用 **read_msg()**，读取一条命令，解析 JSON。

对于 bolts 来说，有少许不同。
  
```
01
while(true) {
02
    $msg = read_msg();
03
    $tuple = json_decode($msg, true, 512, JSON_BIGINT_AS_STRING);
04
    if (!empty($tuple["id"])) {
05
        if (isPrime($tuple["tuple"][0])) {
06
            storm_emit(array($tuple["tuple"][0]));
07
        }
08
        storm_ack($tuple["id"]);
09
    }
10
}  
```   

循环的从标准输入读取元组。解析读取每一条 JSON 消息，判断它是不是一个元组，如果是，再检查它是不是一个素数，如果是素数再次分发一个元组，否则就忽略掉，最后不论如何都要确认成功。
  
**NOTE**：在 **json\_decode** 函数中使用的 **JSON\_BIGINT\_AS\_STRING** 是为了解决一个在JAVA 和 PHP 之间的数据转换问题。JAVA发送的一些很大的数字，在 PHP 中会丢失精度，这样就会导致问题。为了避开这个问题，告诉PHP把大数字当作字符串处理，并在 JSON 消息中输出数字时不使用双引号。PHP5.4.0 或更高版本要求使用这个参数。
 
**emit**，**ack**，**fail**，以及 **log** 消息都是如下结构：

**emit**
  
```
{
    "command": "emit",
    "tuple": ["foo", "bar"]
}  
```  

其中的数组包含了你分发的元组数据。

**ack**
  
```
{
    "command": "ack",
    "id": 123456789
}  
```  

其中的 id 就是你处理的元组的 ID。  

**fail**
  
```
{
    "command": "fail",
    "id": 123456789
}   
```  

与 **ack**（译者注：原文是 **emit** 从上下 JSON 的内容和每个方法的功能上判断此处就是 **ack**，可能是排版错误）相同，其中 id 就是你处理的元组 ID。  
  
**log**
  
```
{
    "command": "log",
    "msg": "some message to be logged by storm."
}   
```  

下面是完整的的 PHP 代码。
  
```
001
//你的spout：
002
<?php
003
function read_msg() {
004
    $msg = "";
005
    while(true) {
006
        $l = fgets(STDIN);
007
        $line = substr($l,0,-1);
008
        if ($line=="end") {
009
            break;
010
        }
011
        $msg = "$msg$line\n";
012
    }
013
    return substr($msg, 0, -1);
014
}
015
function write_line($line) {
016
    echo("$line\n");
017
}
018
function storm_emit($tuple) {
019
    $msg = array("command" => "emit", "tuple" => $tuple);
020
    storm_send($msg);
021
}
022
function storm_send($json) {
023
    write_line(json_encode($json));
024
    write_line("end");
025
}
026
function storm_sync() {
027
    storm_send(array("command" => "sync"));
028
}
029
function storm_log($msg) {
030
    $msg = array("command" => "log", "msg" => $msg);
031
    storm_send($msg);
032
    flush();
033
}
034
$config = json_decode(read_msg(), true);
035
$heartbeatdir = $config['pidDir'];
036
$pid = getmypid();
037
fclose(fopen("$heartbeatdir/$pid", "w"));
038
storm_send(["pid"=>$pid]);
039
flush();
040
$from = intval($argv[1]);
041
$to = intval($argv[2]);
042
while(true) {
043
    $msg = read_msg();
044
    $cmd = json_decode($msg, true);
045
    if ($cmd['command']=='next') {
046
        if ($from<$to) {
047
            storm_emit(array("$from"));
048
            $task_ids = read_msg();
049
            $from++;
050
        } else {
051
            sleep(1);
052
        }
053
    }
054
    storm_sync();
055
}
056
?>
057
//你的bolt:
058
<?php
059
function isPrime($number) {
060
    if ($number < 2) {
061
        return false;
062
    }
063
    if ($number==2) {
064
        return true;
065
    }
066
    for ($i=2; $i<=$number-1; $i++) {
067
        if ($number % $i == 0) {
068
            return false;
069
        }
070
    }
071
    return true;
072
}
073
function read_msg() {
074
    $msg = "";
075
    while(true) {
076
        $l = fgets(STDIN);
077
        $line = substr($l,0,-1);
078
        if ($line=="end") {
079
            break;
080
        }
081
        $msg = "$msg$line\n";
082
    }
083
    return substr($msg, 0, -1);
084
}
085
function write_line($line) {
086
    echo("$line\n");
087
}
088
function storm_emit($tuple) {
089
    $msg = array("command" => "emit", "tuple" => $tuple);
090
    storm_send($msg);
091
}
092
function storm_send($json) {
093
    write_line(json_encode($json));
094
    write_line("end");
095
}
096
function storm_ack($id) {
097
    storm_send(["command"=>"ack", "id"=>"$id"]);
098
}
099
function storm_log($msg) {
100
    $msg = array("command" => "log", "msg" => "$msg");
101
    storm_send($msg);
102
}
103
$config = json_decode(read_msg(), true);
104
$heartbeatdir = $config['pidDir'];
105
$pid = getmypid();
106
fclose(fopen("$heartbeatdir/$pid", "w"));
107
storm_send(["pid"=>$pid]);
108
flush();
109
while(true) {
110
    $msg = read_msg();
111
    $tuple = json_decode($msg, true, 512, JSON_BIGINT_AS_STRING);
112
    if (!empty($tuple["id"])) {
113
        if (isPrime($tuple["tuple"][0])) {
114
            storm_emit(array($tuple["tuple"][0]));
115
        }
116
        storm_ack($tuple["id"]);
117
    }
118
}
119
?>  
```  

**NOTE**：需要重点指出的是，应当把所有的脚本文件保存在你的工程目录下的一个名为**multilang/resources** 的子目录中。这个子目录被包含在发送给工人进程的 jar 文件中。如果你不把脚本包含在这个目录中，Storm 就不能运行它们，并抛出一个错误。 