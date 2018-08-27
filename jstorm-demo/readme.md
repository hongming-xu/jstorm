本文帮助读者快速实现一个JStorm 例子

[Example 源码](https://github.com/alibaba/jstorm/tree/master/example)

#  最简单的JStorm例子分为4个步骤：
## 生成Topology

```java
Map conf = new HashMap();
//topology所有自定义的配置均放入这个Map

TopologyBuilder builder = new TopologyBuilder();
//创建topology的生成器

int spoutParal = get("spout.parallel", 1);
//获取spout的并发设置

SpoutDeclarer spout = builder.setSpout(SequenceTopologyDef.SEQUENCE_SPOUT_NAME,
                new SequenceSpout(), spoutParal);
//创建Spout， 其中new SequenceSpout() 为真正spout对象，SequenceTopologyDef.SEQUENCE_SPOUT_NAME 为spout的名字，注意名字中不要含有空格

int boltParal = get("bolt.parallel", 1);
//获取bolt的并发设置

BoltDeclarer totalBolt = builder.setBolt(SequenceTopologyDef.TOTAL_BOLT_NAME, new TotalCount(),
                boltParal).shuffleGrouping(SequenceTopologyDef.SEQUENCE_SPOUT_NAME);
//创建bolt， SequenceTopologyDef.TOTAL_BOLT_NAME 为bolt名字，TotalCount 为bolt对象，boltParal为bolt并发数，
//shuffleGrouping（SequenceTopologyDef.SEQUENCE_SPOUT_NAME）， 
//表示接收SequenceTopologyDef.SEQUENCE_SPOUT_NAME的数据，并且以shuffle方式，
//即每个spout随机轮询发送tuple到下一级bolt中

int ackerParal = get("acker.parallel", 1);
Config.setNumAckers(conf, ackerParal);
//设置表示acker的并发数

int workerNum = get("worker.num", 10);
conf.put(Config.TOPOLOGY_WORKERS, workerNum);
//表示整个topology将使用几个worker

conf.put(Config.STORM_CLUSTER_MODE, "distributed");
//设置topolog模式为分布式，这样topology就可以放到JStorm集群上运行

StormSubmitter.submitTopology(streamName, conf,
                builder.createTopology());
//提交topology
```
# IRichSpout
IRichSpout 为最简单的Spout接口

```java
 IRichSpout{

    @Override
    public void open(Map conf, TopologyContext context, SpoutOutputCollector collector) {
    }

    @Override
    public void close() {
    }

    @Override
    public void activate() {
    }

    @Override
    public void deactivate() {
    }

    @Override
    public void nextTuple() {
    }

    @Override
    public void ack(Object msgId) {
    }

    @Override
    public void fail(Object msgId) {
    }

    @Override
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
    }

    @Override
    public Map<String, Object> getComponentConfiguration() {
        return null;
    }

```

其中注意：
* spout对象必须是继承Serializable， 因此要求spout内所有数据结构必须是可序列化的
* spout可以有构造函数，但构造函数只执行一次，是在提交任务时，创建spout对象，因此在task分配到具体worker之前的初始化工作可以在此处完成，一旦完成，初始化的内容将携带到每一个task内（因为提交任务时将spout序列化到文件中去，在worker起来时再将spout从文件中反序列化出来）。
* open是当task起来后执行的初始化动作
* close是当task被shutdown后执行的动作
* activate 是当task被激活时，触发的动作
* deactivate 是task被deactive时，触发的动作
* nextTuple 是spout实现核心， nextuple完成自己的逻辑，即每一次取消息后，用collector 将消息emit出去。
* ack， 当spout收到一条ack消息时，触发的动作，详情可以参考 [ack机制](https://github.com/alibaba/jstorm/wiki/Ack-%E6%9C%BA%E5%88%B6)
* fail， 当spout收到一条fail消息时，触发的动作，详情可以参考 [ack机制](https://github.com/alibaba/jstorm/wiki/Ack-%E6%9C%BA%E5%88%B6)
* declareOutputFields， 定义spout发送数据，每个字段的含义
* getComponentConfiguration 获取本spout的component 配置

# Bolt
```java
IRichBolt {

    @Override
    public void prepare(Map stormConf, TopologyContext context, OutputCollector collector) {
    }

    @Override
    public void execute(Tuple input) {
    }

    @Override
    public void cleanup() {
    }

    @Override
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
    }

    @Override
    public Map<String, Object> getComponentConfiguration() {
        return null;
    }

}
```

其中注意：
* bolt对象必须是继承Serializable， 因此要求spout内所有数据结构必须是可序列化的
* bolt可以有构造函数，但构造函数只执行一次，是在提交任务时，创建bolt对象，因此在task分配到具体worker之前的初始化工作可以在此处完成，一旦完成，初始化的内容将携带到每一个task内（因为提交任务时将bolt序列化到文件中去，在worker起来时再将bolt从文件中反序列化出来）。
* prepare是当task起来后执行的初始化动作
* cleanup是当task被shutdown后执行的动作
* execute是bolt实现核心， 完成自己的逻辑，即接受每一次取消息后，处理完，有可能用collector 将产生的新消息emit出去。
** 在executor中，当程序处理一条消息时，需要执行collector.ack， 详情可以参考 [ack机制](https://github.com/alibaba/jstorm/wiki/Ack-%E6%9C%BA%E5%88%B6)
** 在executor中，当程序无法处理一条消息时或出错时，需要执行collector.fail ，详情可以参考 [ack机制](https://github.com/alibaba/jstorm/wiki/Ack-%E6%9C%BA%E5%88%B6)
* declareOutputFields， 定义bolt发送数据，每个字段的含义
* getComponentConfiguration 获取本bolt的component 配置

# 编译
在Maven中配置
```xml
  <dependency>
    <groupId>com.alibaba.jstorm</groupId>
    <artifactId>jstorm-core</artifactId>
    <version>${jstorm.version}</version>
    <scope>provided</scope>
  </dependency>
```


# 提交jar
jstorm jar xxxxxx.jar com.alibaba.xxxx.xx parameter

* xxxx.jar 为打包后的jar
* com.alibaba.xxxx.xx 为入口类，即提交任务的类
* parameter即为提交参数


# 参考文档
(1)JStorm Storm 上手demo
https://blog.csdn.net/jiankunking/article/details/70231744
(2)应用例子
https://github.com/alibaba/jstorm/wiki/%E5%BA%94%E7%94%A8%E4%BE%8B%E5%AD%90


