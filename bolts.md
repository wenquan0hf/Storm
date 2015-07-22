# Bolts  
  
正如你已经看到的，bolts 是一个 Storm 集群中的关键组件。你将在这一章学到 bolt 生命周期，一些 bolt 设计策略，以及几个有关这些内容的例子。

## Bolt 生命周期  

Bolt 是这样一种组件，它把元组作为输入，然后产生新的元组作为输出。实现一个 bolt 时，通常需要实现 **IRichBolt** 接口。Bolts 对象由客户端机器创建，序列化为拓扑，并提交给集群中的主机。然后集群启动工人进程反序列化 bolt，调用 prepare****，最后开始处理元组。

**NOTE**:要创建一个 bolt 对象，它通过构造器参数初始化成员属性，bolt 被提交到集群时，这些属性值会随着一起序列化。

## Bolt 结构  

Bolts拥有如下方法：
  
```
declareOutputFields(OutputFieldsDeclarer declarer)
    为bolt声明输出模式
prepare(java.util.Map stormConf, TopologyContext context, OutputCollector collector)
    仅在bolt开始处理元组之前调用
execute(Tuple input)
    处理输入的单个元组
cleanup()
    在bolt即将关闭时调用  
```  

下面看一个例子，在这个例子中 bolt 把一句话分割成单词列表：
  
```
class SplitSentence implements IRichBolt {
    private OutputCollector collector;
    publlic void prepare(Map conf, TopologyContext context, OutputCollector collector) {
        this.collector = collector;
    }

    public void execute(Tuple tuple) {
        String sentence = tuple.getString(0);
        for(String word : sentence.split(" ")) {
            collector.emit(new Values(word));
        }
    }

    public void cleanup(){}

    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("word"));
    }
}  
```  

正如你所看到的，这是一个很简单的 bolt。值得一提的是在这个例子里，没有消息担保。这就意味着，如果 bolt 因为某些原因丢弃了一些消息——不论是因为 bolt 挂了，还是因为程序故意丢弃的——生成这条消息的 spout 不会收到任何通知，任何其它的 spouts 和 bolts 也不会收到。

然而在许多情况下，你想确保消息在整个拓扑范围内都被处理过了。

## 可靠的 bolts 和不可靠的 bolts  

正如前面所说的，Storm 保证通过 spout 发送的每条消息会得到所有 bolt 的全面处理。基于设计上的考虑，这意味着你要自己决定你的 bolts 是否保证这一点。

拓扑是一个树型结构，消息（元组）穿过其中一条或多条分支。树上的每个节点都会调用 **ack(tuple)** 或 **fail(tuple)**，Storm 因此知道一条消息是否失败了，并通知那个/那些制造了这些消息的 spout(s)。既然一个 Storm 拓扑运行在高度并行化的环境里，跟踪始发 spout 实例的最好方法就是在消息元组内包含一个始发 spout 引用。这一技巧称做锚定(译者注：原文为Anchoring)。修改一下刚刚讲过的 SplitSentence，使它能够确保消息都被处理了。
  
```
class SplitSentence implenents IRichBolt {
    private OutputCollector collector;

    public void prepare(Map conf, TopologyContext context, OutputCollector collector) {
        this.collector = collector;
    }

    public void execute(Tuple tuple) {
        String sentence = tuple.getString(0);
        for(String word : sentence.split(" ")) {
            collector.emit(tuple, new Values(word));
        }
        collector.ack(tuple);
    }

    public void cleanup(){}

    public void declareOutputFields(OutputFieldsDeclarer declarer){
        declar.declare(new Fields("word"));
    }
}  
```  

锚定发生在调用 **collector.emit()** 时。正如前面提到的，Storm 可以沿着元组追踪到始发spout。**collector.ack(tuple)** 和 **collector.fail(tuple)**会告知 spout 每条消息都发生了什么。当树上的每条消息都已被处理了，Storm 就认为来自 spout 的元组被全面的处理了。如果一个元组没有在设置的超时时间内完成对消息树的处理，就认为这个元组处理失败。默认超时时间为30秒。

NOTE:你可以通过修改Config.TOPOLOGY_MESSAGE_TIMEOUT修改拓扑的超时时间。

当然了spout需要考虑消息的失败情况，并相应的重试或丢弃消息。

**NOTE**:你处理的每条消息要么是确认的（译者注：collector.ack()）要么是失败的（译者注：collector.fail()）。Storm 使用内存跟踪每个元组，所以如果你不调用这两个方法，该任务最终将耗尽内存。

## 多数据流  

一个 bolt 可以使用 **emit(streamId, tuple)** 把元组分发到多个流，其中参数 **streamId** 是一个用来标识流的字符串。然后，你可以在 **TopologyBuilder** 决定由哪个流订阅它。

## 多锚定  

为了用 bolt 连接或聚合数据流，你需要借助内存缓冲元组。为了在这一场景下确保消息完成，你不得不把流锚定到多个元组上。可以向 **emit** 方法传入一个元组列表来达成目的。
  
```
...
List anchors = new ArrayList();
anchors.add(tuple1);
anchors.add(tuple2);
collector.emit(anchors, values);
...  
```  

通过这种方式，bolt 在任意时刻调用 **ack** 或 **fail** 方法，都会通知消息树，而且由于流锚定了多个元组，所有相关的 spout 都会收到通知。

## 使用 IBasicBolt 自动确认  

你可能已经注意到了，在许多情况下都需要消息确认。简单起见，Storm 提供了另一个用来实现bolt 的接口，IBasicBolt。对于该接口的实现类的对象，会在执行 **execute** 方法之后自动调用 **ack** 方法。
  
```
class SplitSentence extends BaseBasicBolt {
    public void execute(Tuple tuple, BasicOutputCollector collector) {
        String sentence = tuple.getString(0);
        for(String word : sentence.split(" ")) {
            collector.emit(new Values(word));
        }
}

    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("word"));
    }
}  
```  

**NOTE**:分发消息的 BasicOutputCollector 自动锚定到作为参数传入的元组。