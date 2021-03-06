---
title: Flux
layout: documentation
documentation: true
---

一个框架，为了让创建和部署Apache Storm“流”计算遇上更方便快捷

## 定义
**flux** |fləks| _名词_

1. 流动或者流出的这个动作过程 这个动作或者流出的过程
2. 持续不断的改变
3. 在物理中，表明液体、辐射能或者颗粒在指定区域内的流速
4. 一个混合了固体用来降低其熔点的物质

## 基本原理
当配置很难被编程的时候会发生糟糕的事情。没有人应因为需要修改配置而重新编译或者重新打包一个应用。

## 相关
Flux是一个用来让规定和部署Apache Storm拓扑不那么费劲的框架和一系列工具。

你是否发现你总是重复这样的模式？:

```java

public static void main(String[] args) throws Exception {
    // 返回的逻辑值用来判断我们是否在本地上运行
    // 创建必要的配置选项...
    boolean runLocal = shouldRunLocal();
    if(runLocal){
        LocalCluster cluster = new LocalCluster();
        cluster.submitTopology(name, conf, topology);
    } else {
        StormSubmitter.submitTopology(name, conf, topology);
    }
}
```

像这样的操作会不会更容易一些呢:

```bash
storm jar mytopology.jar org.apache.storm.flux.Flux --local config.yaml
```

或者:

```bash
storm jar mytopology.jar org.apache.storm.flux.Flux --remote config.yaml
```

另一个比较经常提及的麻烦点在于由于写拓扑图常常是和Java代码紧密结合的，所以任何对它的修改都需要重新编译和重新打包拓扑的jar文件。Flux的目标是允许你将你所有的Storm组件打包在一个单独的jar文件中，然后使用另一个文本文件来规定你的拓扑的布局和配置。通过这样的方式，缓解这一个麻烦。

## 特点

 * 安装和部署Storm拓扑（包括Storm core和Microbatch API）简单，而不是用内嵌的配置方法在你的拓扑代码中安装和部署。
 * 支持已有的拓扑代码（如下可见）
 * 通过使用灵活的YAML DSL定义Storm Core API（Spouts/Bolts）。
 * YAML DSL支持大多数的Storm组件 (storm-kafka, storm-hdfs, storm-hbase, 等等.)
 * 对多语言的组件有便捷的支持
 * 为了更简便地在配置/环境间转换，使用了外部属性的置换/过滤（类似于Maven风格的`${variable.name}`置换）。

## 用法

为了使用Flux，把它添加到依赖包中，然后把你所有的Storm组件打包成一个很大的jar文件，再然后创建一个YAML文件来定义你的拓扑（下面有YAML配置选项）。

### 通过源码来构建
使用Flux最简单的方法就是将它作为Maven依赖包添加到项目中，如下面描述的那样。

如果你要从源代码中创建Flux并进行单元/集成的测试，你需要在你的系统上按照如下操作来安装：

* Python 2.6.x or later
* Node.js 0.10.x or later

#### 创建能够使用单元测试的命令：

```
mvn clean install
```

#### 创建不能够使用单元测试的命令：
如果你希望在不安装Python和Node.js的情况下构建Flux，你可以跳过这个单元测试：

```
mvn clean install -DskipTests=true
```

需要注意，如果你打算使用Flux来给远程的簇部署拓扑，你仍然需要安装Python，因为Apche Storm要求这么做。

#### 创建能够使用集成测试的命令：

```
mvn clean install -DskipIntegration=false
```


### 和Maven一起打包
为了保证Flux能对你的Storm组件有效，你需要把Flux当做依赖包添加，这样才能让它包含在Storm的拓扑jar文件中。这个可以通过Maven shade插件（推荐）或者Maven assembly插件（不推荐）来完成。

#### Flux Maven依赖包
Flux现在的版本可以在以下的合作方的Maven中心获得：
```xml
<dependency>
    <groupId>org.apache.storm</groupId>
    <artifactId>flux-core</artifactId>
    <version>${storm.version}</version>
</dependency>
```

使用shell的spouts和bolt要求附加的Flux Wrappers库：
```xml
<dependency>
    <groupId>org.apache.storm</groupId>
    <artifactId>flux-wrappers</artifactId>
    <version>${storm.version}</version>
</dependency>
```

#### 创建一个允许使用Flux的拓扑JAR文件
下述的例子阐述了如何配合Maven shade插件使用Flux：

 ```xml
<!-- 在shaded jar文件中包含FLux和用户依赖包 -->
<dependencies>
    <!-- Flux include -->
    <dependency>
        <groupId>org.apache.storm</groupId>
        <artifactId>flux-core</artifactId>
        <version>${storm.version}</version>
    </dependency>
    <!-- Flux Wrappers include -->
    <dependency>
        <groupId>org.apache.storm</groupId>
        <artifactId>flux-wrappers</artifactId>
        <version>${storm.version}</version>
    </dependency>

    <!-- 在这里添加用户依赖包... -->

</dependencies>
<!-- 创建一个包括所有依赖包的大大的jar文件 -->
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-shade-plugin</artifactId>
            <version>1.4</version>
            <configuration>
                <createDependencyReducedPom>true</createDependencyReducedPom>
            </configuration>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>shade</goal>
                    </goals>
                    <configuration>
                        <transformers>
                            <transformer
                                    implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer"/>
                            <transformer
                                    implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                <mainClass>org.apache.storm.flux.Flux</mainClass>
                            </transformer>
                        </transformers>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
 ```

### 部署和运行Flux拓扑
一旦你的拓扑组件和Flux的依赖包一起打包后，你就可以通过使用 `storm jar` 命令在本地或者远端运行不同的拓扑。比如说，如果你的大大的jar文件命名为 `myTopology-0.1.0-SNAPSHOT.jar` ，你可以使用以下的命令在本地运行它：


```bash
storm jar myTopology-0.1.0-SNAPSHOT.jar org.apache.storm.flux.Flux --local my_config.yaml

```

### 命令行的参数选项
```
usage: storm jar <my_topology_uber_jar.jar> org.apache.storm.flux.Flux
             [options] <topology-config.yaml>
 -d,--dry-run                 不运行/部署这个拓扑，仅仅是构建、验证和打印这个拓扑的相关信息。
 -e,--env-filter              执行环境变量的替换。 以形式为 `${ENV-[NAME]}` 定义的替换关键字将会替换 `NAME` 对应的环境变量值。
 -f,--filter <file>           执行属性替换。使用一个指定的文件作为属性的源，然后形式为 {$[property name]} 的替换关键字将会替换在这个属性文件中的值。.
 -i,--inactive                部署但是不激活这个拓扑。
 -l,--local                   在local的模式下运行拓扑。
 -n,--no-splash               抑制版权页的输出。
 -q,--no-detail               抑制拓扑详情的输出。
 -r,--remote                  将拓扑部署到远端的簇。
 -R,--resource                使用提供的路径来作为classpath的源文件以代替文件。
 -s,--sleep <ms>              当在本地运行时，在killing一个拓扑和关闭本地簇之前需要sleep的时间（以ms为单位）
 -z,--zookeeper <host:port>   当以local模式运行时，使用ZooKeeper的特定<host:port>而不是同进程的的ZooKeeper。（要求在Storm的0.9.3或之后的版本）  
```

**注意：** Flux为了避免在使用到 `storm` 命令时产生命令行开关冲突，所以允许任何其他的命令行开关来表达 `storm` 这一命令。

举例来说，你可以使用`storm`的命令开关`-c`来覆盖拓扑配置性能。下述举例的命令就可以运行Flux并且覆盖`nimbus.seeds`这一配置：

```bash
storm jar myTopology-0.1.0-SNAPSHOT.jar org.apache.storm.flux.Flux --remote my_config.yaml -c 'nimbus.seeds=["localhost"]'
```

### 输出例子
```
███████╗██╗     ██╗   ██╗██╗  ██╗
██╔════╝██║     ██║   ██║╚██╗██╔╝
█████╗  ██║     ██║   ██║ ╚███╔╝
██╔══╝  ██║     ██║   ██║ ██╔██╗
██║     ███████╗╚██████╔╝██╔╝ ██╗
╚═╝     ╚══════╝ ╚═════╝ ╚═╝  ╚═╝
+-         Apache Storm        -+
+-  data FLow User eXperience  -+
Version: 0.3.0
Parsing file: /Users/hsimpson/Projects/donut_domination/storm/shell_test.yaml
---------- TOPOLOGY DETAILS ----------
Name: shell-topology
--------------- SPOUTS ---------------
sentence-spout[1](org.apache.storm.flux.wrappers.spouts.FluxShellSpout)
---------------- BOLTS ---------------
splitsentence[1](org.apache.storm.flux.wrappers.bolts.FluxShellBolt)
log[1](org.apache.storm.flux.wrappers.bolts.LogInfoBolt)
count[1](org.apache.storm.testing.TestWordCounter)
--------------- STREAMS ---------------
sentence-spout --SHUFFLE--> splitsentence
splitsentence --FIELDS--> count
count --SHUFFLE--> log
--------------------------------------
Submitting topology: 'shell-topology' to remote cluster...
```

## YAML配置
Flux拓扑在YAML文件中被顶迎来描述一个拓扑。一个Flux拓扑定义由以下几项组成：

  1. 一个拓扑的名字
  2. 一个拓扑组件的列表（被命名为Java对象，用以可以在环境中可以调用）
  3. **第三选项或者第四选项** （一个DSL拓扑定义）:
      * 一个spouts的列表，每一个项通过一个唯一的ID被识别
      * 一个bolts的列表，每一个项通过一个唯一的ID被识别
      * 一个“stream”对象的列表，用来表示spouts和bolts之间的流的元组。
  4. **第三选项或者第四选项** (一个可以创建 `org.apache.storm.generated.StormTopology` 实例的JVM类）：
      * 一个 `topologySource` 定义。



举个例子，这里有一个使用YAML DSL的简单wordcount拓扑：

```yaml
name: "yaml-topology"
config:
  topology.workers: 1

# spout定义
spouts:
  - id: "spout-1"
    className: "org.apache.storm.testing.TestWordSpout"
    parallelism: 1

# bolt定义
bolts:
  - id: "bolt-1"
    className: "org.apache.storm.testing.TestWordCounter"
    parallelism: 1
  - id: "bolt-2"
    className: "org.apache.storm.flux.wrappers.bolts.LogInfoBolt"
    parallelism: 1

# stream定义
streams:
  - name: "spout-1 --> bolt-1" #name暂时未用上（可以在logging,UI等中作为placeholder）
    from: "spout-1"
    to: "bolt-1"
    grouping:
      type: FIELDS
      args: ["word"]

  - name: "bolt-1 --> bolt2"
    from: "bolt-1"
    to: "bolt-2"
    grouping:
      type: SHUFFLE


```
## 属性替换/过滤
对于开发者而言，想要简单地转换配置是常有的事，例如在开发环境和生产环境中转换部署。这可以通过使用分开的YAML配置文件来实现，但是这个方法会导致配置文件中多出一些多余的重复内容，尤其是在一些Storm拓扑没有改变但是配置设置例如主机名，端口和并行性参数改变了的情况。

对于这种情况，Flux提供了属性过滤（properties filtering）方法允许你个给一个 `.properties` 文件赋两个具体的值，并且让他们在 `.yaml` 文件被解析前被替代。

为了实现属性过滤功能，使用 `--filter` 命令行选项，并且具体制定一个 `.properties` 文件。举个例子，如果你像这样调用flux：

```bash
storm jar myTopology-0.1.0-SNAPSHOT.jar org.apache.storm.flux.Flux --local my_config.yaml --filter dev.properties
```
并且 `dev.properties` 内容如下：

```properties
kafka.zookeeper.hosts: localhost:2181
```

你在这之后就可以通过你 `.yaml` 文件中的属性关键字，使用 `${}` 语法来引用它：

```yaml
  - id: "zkHosts"
    className: "org.apache.storm.kafka.ZkHosts"
    constructorArgs:
      - "${kafka.zookeeper.hosts}"
```

在这个例子中，Flux可以在YAML内容被解析前使用 `localhost:2181` 来替换 `${kafka.zookeeper.hosts}` 。

### 环境变量替换/过滤Environment Variable Substitution/Filtering
Flux同样也允许环境变量替换。举个例子，如果名为`ZK_HOSTS` 的环境变量名被定义了，你可以通过以下的语法在Flux的YAML文件中引用它：

```
${ENV-ZK_HOSTS}
```

## 组件
组件从本质来说是对象实例，用来在对spouts和bolts的配置选项中获取。如果你对Spring框架很熟悉，这里的组件大概就类比于Spring中的beans

每一个组件都是可被识别的，至少是可以通过一个唯一的标识符（字符串）和一个类名（字符串）。举个例子，以下的例子将会创建一个 `org.apache.storm.kafka.StringScheme` 类的实例作为关键字 `"stringScheme"` 的引用。这里我们假设这个类 `org.apache.storm.kafka.StringScheme` 有一个默认的构造函数。

```yaml
components:
  - id: "stringScheme"
    className: "org.apache.storm.kafka.StringScheme"
```

### 构造函数参数，引用，属性和配置方法

####构造函数参数
为了给一个类的构造函数添加参数，我们添加 `contructorArgs` 这个元素给组件。 `contructorArgs` 是一个列表，其元素是对象。这个列表会被传递给类的构造函数们。以下的这个例子通过调用一个把单个字符串作为参数的构造函数来创建一个对象：

```yaml
  - id: "zkHosts"
    className: "org.apache.storm.kafka.ZkHosts"
    constructorArgs:
      - "localhost:2181"
```

####引用
每一个组件实例都通过一个唯一的id可悲其他组件重复使用。为了引用一个已存在的组件，你需要在使用 `ref` 这个标签的时候指明这个组件的id。

在以下的例子中，一个名为的组件被创建，之后将被作为另一个组件的构造函数的参数被引用：

```yaml
components:
  - id: "stringScheme"
    className: "org.apache.storm.kafka.StringScheme"

  - id: "stringMultiScheme"
    className: "org.apache.storm.spout.SchemeAsMultiScheme"
    constructorArgs:
      - ref: "stringScheme" # component with id "stringScheme" must be declared above.
```
**注意:** 引用只能在对象被声明后使用。

####属性
除去允许在调用构造函数的时候传进不同的参数，Flux同样允许在配置组件的时候使用被声明为 `public` 的类似JavaBean的setter方法和域：

```yaml
  - id: "spoutConfig"
    className: "org.apache.storm.kafka.SpoutConfig"
    constructorArgs:
      # brokerHosts
      - ref: "zkHosts"
      # topic
      - "myKafkaTopic"
      # zkRoot
      - "/kafkaSpout"
      # id
      - "myId"
    properties:
      - name: "ignoreZkOffsets"
        value: true
      - name: "scheme"
        ref: "stringMultiScheme"
```

在上述的例子中，如果声明了 `properties` ，Flux将会在 `SpoutConfig` 中找一个名字为 `setIgnoreZkOffsets(boolean b)` 的函数并试图调用它。如果这样的一个setter函数没有找到，Flux就会找一个公有的叫 `ignoreZkOffsets` 的实例变量并且将它进行设置。

引用也可能被作为属性值来使用。

####配置方法
从概念上来说，配置方法可能类似于属性和构造函数的参数 —— 他们允许一个对象在创建后可以调用任意的方法。对于有一些类，它们没有暴露JavaBean的方法或者没有能够将整个对象都配置好的构造函数，配置方法对这种类就十分有用。一些常见的例子包括了哪些使用构造器模式来配置/整合的类。

接下来的YAML例子创建了一个bolt并且通过几个方法进行了配置：

```yaml
bolts:
  - id: "bolt-1"
    className: "org.apache.storm.flux.test.TestBolt"
    parallelism: 1
    configMethods:
      - name: "withFoo"
        args:
          - "foo"
      - name: "withBar"
        args:
          - "bar"
      - name: "withFooBar"
        args:
          - "foo"
          - "bar"
```

对应方法的标识如下：

```java
    public void withFoo(String foo);
    public void withBar(String bar);
    public void withFooBar(String foo, String bar);
```

传递给配置方法的参数和构造函数中的参数作用一样，并且也支持引用。

### 在沟早期的参数，引用，属性和配制方法中使用Java的 `enum`s in Contructor Arguments, References, Properties and Configuration Methods
你可以在Flux YAML文件中轻松通过引用 `enum` 的名字将其值作为参数。

比如， [Storm's HDFS 模块]() 包含了以下 `enum` 的定义（为了简洁而简化过）：

```java
public static enum Units {
    KB, MB, GB, TB
}
```

 `org.apache.storm.hdfs.bolt.rotation.FileSizeRotationPolicy` 这个类有如下的构造器：

```java
public FileSizeRotationPolicy(float count, Units units)

```
以下的Flux `component` 定义可以被用来调用这个构造器：

```yaml
  - id: "rotationPolicy"
    className: "org.apache.storm.hdfs.bolt.rotation.FileSizeRotationPolicy"
    constructorArgs:
      - 5.0
      - MB
```

上述的定义和下面的Java代码从功能上来说是一样的：

```java
// rotate files when they reach 5MB
FileRotationPolicy rotationPolicy = new FileSizeRotationPolicy(5.0f, Units.MB);
```

## 拓扑配置
 `config` 这个区段仅仅是Storm拓扑配置参数的一个图，将会作为 `org.apache.storm.Config` 类的实例传给 `org.apache.storm.StormSubmitter` ：

```yaml
config:
  topology.workers: 4
  topology.max.spout.pending: 1000
  topology.message.timeout.secs: 30
```

# 已经存在的拓扑
如果你有已经存在的Storm拓扑，你仍然可以用Flux来部署/运行/测试它们。这个特点允许你按照已有的拓扑类来改变Flux构造参数，引用，属性和拓扑配置声明。

使用已有拓扑类最简单的方法就是通过下面的方法定义一个名为 `getTopology()` 的实例方法：

```java
public StormTopology getTopology(Map<String, Object> config)
```
或者：

```java
public StormTopology getTopology(Config config)
```

你接下来就可以使用YAML来部署你的拓扑：

```yaml
name: "existing-topology"
topologySource:
  className: "org.apache.storm.flux.test.SimpleTopology"
```

如果你想用来作为拓扑源的类有一个不同的方法名（比如不叫），那么你可以把它重写：

```yaml
name: "existing-topology"
topologySource:
  className: "org.apache.storm.flux.test.SimpleTopology"
  methodName: "getTopologyWithDifferentMethodName"
```

__注意：__ 这个指定的方法必须接受一个单一的类型是 `java.util.Map<String, Object>` 或者 `org.apache.storm.Config` 的类，然后返回一个 `org.apache.storm.generated.StormTopology` 对象。

# YAML DSL
## Spouts 和 Bolts
Spout和Bolts是在YAML配置中各自的区域中被配置。Spout和Bolt的定义是 `组件（component）` 定义的拓展。在组件的基础上添加了 `并行度（parallelism）`参数，用于当一个拓扑被部署的时候设置组件的并行度。

因为spout和bolt定义继承了 `组件（component）` ，所以它们也支持构造函数参数，引用，属性。Because spout and bolt definitions extendthey support constructor arguments, references, and properties as
well.

Shell spout的例子：

```yaml
spouts:
  - id: "sentence-spout"
    className: "org.apache.storm.flux.wrappers.spouts.FluxShellSpout"
    # shell spout constructor takes 2 arguments: String[], String[]
    constructorArgs:
      # command line
      - ["node", "randomsentence.js"]
      # output fields
      - ["word"]
    parallelism: 1
```

Kafka spout的例子：

```yaml
components:
  - id: "stringScheme"
    className: "org.apache.storm.kafka.StringScheme"

  - id: "stringMultiScheme"
    className: "org.apache.storm.spout.SchemeAsMultiScheme"
    constructorArgs:
      - ref: "stringScheme"

  - id: "zkHosts"
    className: "org.apache.storm.kafka.ZkHosts"
    constructorArgs:
      - "localhost:2181"

# 可选的kafka配置
#  - id: "kafkaConfig"
#    className: "org.apache.storm.kafka.KafkaConfig"
#    constructorArgs:
#      # brokerHosts
#      - ref: "zkHosts"
#      # topic
#      - "myKafkaTopic"
#      # clientId (optional)
#      - "myKafkaClientId"

  - id: "spoutConfig"
    className: "org.apache.storm.kafka.SpoutConfig"
    constructorArgs:
      # brokerHosts
      - ref: "zkHosts"
      # topic
      - "myKafkaTopic"
      # zkRoot
      - "/kafkaSpout"
      # id
      - "myId"
    properties:
      - name: "ignoreZkOffsets"
        value: true
      - name: "scheme"
        ref: "stringMultiScheme"

config:
  topology.workers: 1

# spout definitions
spouts:
  - id: "kafka-spout"
    className: "org.apache.storm.kafka.KafkaSpout"
    constructorArgs:
      - ref: "spoutConfig"

```

Bolt 例子：

```yaml
# bolt definitions
bolts:
  - id: "splitsentence"
    className: "org.apache.storm.flux.wrappers.bolts.FluxShellBolt"
    constructorArgs:
      # command line
      - ["python", "splitsentence.py"]
      # output fields
      - ["word"]
    parallelism: 1
    # ...

  - id: "log"
    className: "org.apache.storm.flux.wrappers.bolts.LogInfoBolt"
    parallelism: 1
    # ...

  - id: "count"
    className: "org.apache.storm.testing.TestWordCounter"
    parallelism: 1
    # ...
```
## Streams and Stream Groupings
Flux中的“流”被表示为一列在Spouts和Bolts之间的“连接”（如图的边，数据流等），在连接定义的同时有一个关联的“分组”定义。

一个“流”定义有如下的属性：

**`name`:** 一个“连接”的名字（可选的，当下不会马上使用）

**`from`:** 作为源头的Spout或者Bolt的 `id`（类似于出版商）

**`to`:** 作为目的地的Spout或者Bolt的 `id` （类似于订阅者）

**`grouping`:** 为了“流”而产生的“流分组”定义

一个“分组”定义有以下的属性：

**`type`:** 分组的类型。下列值中任选一个 `ALL`、`CUSTOM`、`DIRECT`、`SHUFFLE`、`LOCAL_OR_SHUFFLE`、`FIELDS`、`GLOBAL`、或者 `NONE`。

**`streamId`:** Storm “流”的ID（可选的，如果没有指定则会使用默认流）

**`args`:** 当 `type` 的值为 `FIELDS` 时，一系列域的名字。

**`customClass`** 当 `type` 的值为 `CUSTOM` 时，自定义的“分组”类实例的定义

如下例的 `流（streams）` 的定义案例建立起了一个如下的线路拓扑：

```
    kafka-spout --> splitsentence --> count --> log
```


```yaml
# 流（stream）定义
# “流”的定义定了在spouts和bolts之间的“连接”。
# 注意这样的“连接”可能是循环的
# 自定义的“流分组”也被支持

streams:
  - name: "kafka --> split" # name isn't used (placeholder for logging, UI, etc.)
    from: "kafka-spout"
    to: "splitsentence"
    grouping:
      type: SHUFFLE

  - name: "split --> count"
    from: "splitsentence"
    to: "count"
    grouping:
      type: FIELDS
      args: ["word"]

  - name: "count --> log"
    from: "count"
    to: "log"
    grouping:
      type: SHUFFLE
```

### 自定义“流分组”
自定义的流分组是通过设置分组的类型为 `CUSTOM` 并且定义一个 `customClass` 参数，该参数告诉Flux如何实例化一个自定义类。这个 `customClass` 定义继承自 `组件（component）`，所以它也支持构造函数参数，引用和属性。

如下的例子创建了一个”流“，并且使用了一个类型为 `org.apache.storm.testing.NGrouping` 的自定义“流分组”类。

```yaml
  - name: "bolt-1 --> bolt2"
    from: "bolt-1"
    to: "bolt-2"
    grouping:
      type: CUSTOM
      customClass:
        className: "org.apache.storm.testing.NGrouping"
        constructorArgs:
          - 1
```

## “包含”和“重写”
FLux允许包含其他YAML文件的内容，并且把它们当做在一个文件中定义的一样。可以包含文件或者路径源文件。

“包含”是通过一系列的maps来指定的：

```yaml
includes:
  - resource: false
    file: "src/test/resources/configs/shell_test.yaml"
    override: false
```

如果 `resource` 的值为 `true`，“包含”将会从 `file` 这个属性值中来加载路径源文件，否则它会被当做是普通的文件。 

`override` 属性控制着“包含”要如何影响定义在当前文件中的值。如果 `override` 的值是
`true`，那么file值将会替代现在的文件被解析。如果 `override` 的值是
`false`，那么当前文件正在解析的值会有优先权，并且解析器会拒绝将它们替换掉。

**注意：** “包含”现今不是循环的，包含文件中的包含将会被忽视。


## 基本的Word Count例子

这个例子使用了在JavaScript中实现的spout，Python中实现的bolt，和另一个在Java中实现的bolt。

拓扑 YAML 配置：

```yaml
---
name: "shell-topology"
config:
  topology.workers: 1

# spout 定义
spouts:
  - id: "sentence-spout"
    className: "org.apache.storm.flux.wrappers.spouts.FluxShellSpout"
    # shell spout constructor takes 2 arguments: String[], String[]
    constructorArgs:
      # command line
      - ["node", "randomsentence.js"]
      # output fields
      - ["word"]
    parallelism: 1

# bolt 定义
bolts:
  - id: "splitsentence"
    className: "org.apache.storm.flux.wrappers.bolts.FluxShellBolt"
    constructorArgs:
      # command line
      - ["python", "splitsentence.py"]
      # output fields
      - ["word"]
    parallelism: 1

  - id: "log"
    className: "org.apache.storm.flux.wrappers.bolts.LogInfoBolt"
    parallelism: 1

  - id: "count"
    className: "org.apache.storm.testing.TestWordCounter"
    parallelism: 1

#stream 定义
# “流”定义定义了在spouts和bolts之间的连接
# 注意“连接”可能是循环的
# 自定义“流分组”也是被支持的

streams:
  - name: "spout --> split" # name没有被使用（ 是logging，UI等中的占位符）
    from: "sentence-spout"
    to: "splitsentence"
    grouping:
      type: SHUFFLE

  - name: "split --> count"
    from: "splitsentence"
    to: "count"
    grouping:
      type: FIELDS
      args: ["word"]

  - name: "count --> log"
    from: "count"
    to: "log"
    grouping:
      type: SHUFFLE
```


## Micro-Batching (Trident) API 支持
虽然目前Flux DSL只支持核心Storm API（the COre Storm API），但是对Storm的micro-batching API的支持正在计划中。

为了和Trident拓扑一起使用Flux，在你的YAML配置中定义一个拓扑的getter方法和引用：
```yaml
name: "my-trident-topology"

config:
  topology.workers: 1

topologySource:
  className: "org.apache.storm.flux.test.TridentTopologySource"
  # FLux将会寻找 "getTopology"方法, 这个会用来重写之前那个
  methodName: "getTopologyWithDifferentMethodName"
```
