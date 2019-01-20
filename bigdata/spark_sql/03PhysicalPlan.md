**物理计划是Spark SQL整个查询处理流程的最后一步，与底层平台紧密相关。在此阶段，Spark SQL会对生成的逻辑算子树进一步处理，得到物理算子树，并将LogicalPlan节点及其所包含的各种信息映射成Spark Core计算模型的元素，
如RDD，Transformation，Action等。**
## Spark SQL物理计划概述
从Optimized LogicalPlan传入到Spark SQL物理计划提交并执行，主要经过3个阶段：
```text
Optimized LogicalPlan ---> (SparkPlanner plan ...strategies...)--- Iterator[PhysicalPlan] ---> next()--- SparkPlan ---> prepareForExecution() Seq[Rule[SparkPlan]]--- Prepared SparkPlan
```
>* SparkPlanner将各种物理计划策略作用于对应的LogicalPlan节点上，生成SparkPlan列表(一个LogicalPlan可能产生多种SparkPlan)。
>* 选取最佳的SparkPlan，在2.1版本中实现较为简单，在候选列表中直接用next()方法获取第一个。
>* 提交前进行准备工作，进行一些分区排序方面的处理，确保SparkPlan各节点能够执行。

## SparkPlan简介
在物理算子树中，叶子类型的SparkPlan节点负责创建RDD，每个非叶子类型的SparkPlan等价于在RDD上进行一次Transformation，即通过调用execute()函数转换成新的RDD，最终执行collect()操作触发计算，返回结果给用户。**Transformation过程中除了
对数据操作外，还可能对RDD的分区进行调整。**   
具体来看，SparkPlan的主要功能可以划分为3大块：
>* Metadata和Metric体系：记录元数据与指标信息
>* Partitioning与Ordering体系：在Transformation操作时，会涉及分区与排序的处理
>* 执行操作部分：SparkPlan作为物理计划，支持提交到Spark Core去执行，即SparkPlan的执行操作部分，以execute和executeBroadcast(将数据广播到集群中)方法为主。

Spark SQL中大约包含65种具体的SparkPlan实现，涉及数据源RDD的创建和各种数据处理等，大致分为四类：
>* LeafExecNode类型：物理执行计划中与数据源相关的节点都属于该类型。该类型的SparkPlan负责对初始RDD创建。
>* UnaryExecNode类型： 一元，作用主要是对RDD进行转换操作。
>* BinaryExecfNode类型：二元，除了CoGroupExec之外，其他都是不同类型的Join执行计划。
>* 其他类型的SparkPlan：......

## Metadata与Metrics体系
元数据和指标信息是性能优化的基础，SparkPlan通过Map类型的数据结构来存储相关信息。   
一般情况下，元数据信息Metadata对应Map中的key，value都为字符串类型。主要用于描述数据源的一些基本信息，例如数据文件的格式，存储路径等。  
指标信息Metrics对应Map的key为字符串类型，而value部分是SQLMetrics类型。在Spark执行过程中，Metrics能够记录各种信息，为应用的诊断和优化提供基础。一共有27个SparkPlan重载了该方法。  

## Partioning和Ordering体系
SparkPlan中实现了较为完整的分区与排序操作体系。
Partioning和Ordering体系可以概括为“承前启后”，“承前”体现在对输入数据特性的需求上，requiredChildDistribution和requiredChildOrdering分别规定了当前SparkPlan所需要的数据分布和数据排序方式列表，本事上是对所有子节点输出数据(RDD)的约束。
“启后”体现在对输出数据的操作上，outputPartioning定义了SparkPlan对输出数据(RDD)的分区操作，outputOrdering则定义了每个数据分区的排序方式。  
### Distribution与Partitioning的概念  
两者均被定义为接口，**了解两者关系以及具体实现类**。
#### Distribution  
Distribution定义了查询时，同一个表达式下的不同数据元组在集群各个节点上的分布情况。Distribution描述两种不同粒度的数据特征：
>* 节点间分区信息，数据元组在集群不同的物理节点上是如何分区的。
>* 分区数据内的排序信息，单个分区内数据时如何分布的。  

在2.1中有5种Distribution的实现：
>* UnspecifiedDistribution
>* AllTuples
>* BroadcastDistribution: 广播分布，举个例子：如果时Broadcast类型的Join操作，假设左表做广播，那么requiredChildDistribution得到的列表就是[BroadcastDistribution(mode), UnspecifiedDistribution]。
>* ClusteredDistribution
>* OrderedDistribution

#### Partitioning
Partitioning定义了一个物理算子输出数据的分区方式，描述了SparkPlan中进行分区的操作。Partitioning接口中包含一个成员变量和三个函数来进行分区操作：
>* numPartitions: 
>* satisfies(required:Distribution)
>* compatibleWith(other: Partitioning): 当存在多个子节点时，需要判断不同的子节点的分区操作是否兼容(一般当两个partition能够将相同key的数据分发到相同分区时，才能够兼容)
>* guarantees(other: Partitioning): 如果a.guarantees(b)能够成真，那么任何A进行分区操作所产生的数据行也能够被B产生。避免重分区。  

Partitioning也有多种具体实现，以HashPartitioning为例(在Aggregation和Join操作中被应用)：
```text
case class HashPartitioning(expresssion: Seq[Expression], numPartitions: Int) extends Expression with Partitioning with Unevaluable {
  override def satisfies(required: Distribution): Boolean = required match {
    case UnspecifiedDistribution => true
    case ClusteredDistribution(requiredClusering) => 
      expressions.forall(x => requiredClustering.exists(_.semanticEquals(x)))
    case _ => false
  }
  override def compatibleWith(other: Partitioning): Boolean = other match {
    case o: HashPartitioning => this.semanticEquals(o)
    case _ => false
  }
  override def guarantees(other: Partitioning): Boolean = other match {
    case o: HashPartitioning => this.semanticEquals(o)
    case _ => false
  }
}
```
本例中涉及的几个SparkPlan的分区排序操作：
```text
//数据文件扫描执行算子，物理执行树种的叶子节点。分区排序信息会根据数据文件构造的初始的RDD进行设置。  
//过滤执行算子与列剪裁执行算子，分区与排序的方式仍然沿用其子节点的方式，即不对RDD的分区与排序进行任何的重新操作。  
具体看源码
```
## SparkPlan生成
逻辑计划处理完毕后，会构造SparkPlanner并执行plan()方法对LogicalPlan进行处理，得到对应的物理计划，实际上得到的是一个物理计划列表(Iterator[SparkPlan])。  
```text
QueryPlanner[PhysicalPlan] <-- SparkStrategies <-- SparkPlanner
```
plan()方法实现在QueryPlanner类种。SparkStrategies类本身不提供任何方法，而是在内部提供一批SparkPlanner会用到的各种策略(Strategy)实现。最后，在SparkPlanner层面将这些策略整合在一起，通过plan()方法进行逐个应用。
生成物理计划的实现代码如下所示：
```text
def plan(plan: LogicalPlan): Iterator[PhysicalPlan] = {
  //生成物理计划候选集合，如果集合种存在PlanLater类型的SparkPlan，则通过placeholder中间变量取出对应的LogicalPlan后，递归调用plan()方法，将PlanLater替换为子节点的物理计划。
  val candidates = strategies.iterator.flatMap(_(plan))
  val plans = candiadtes.flatMap { candidate =>
    val placeholders = collectPlaceholders(candidate)
    if (placeholders.isEmpty) {
      Iterator(candidate)
    } else {
      placeholders.iterator.foldLeft(Iterator(candidate)) {
        case (candidatesWithPlaceholders, (placeholder, logicalplan)) => 
          val childPlans = this.plan(logicalPlan)
          candidatesWithPlaceholders.flatMap { candidateWithPlaceholders =>
            childPlans.map { childPlan =>
              candidateWithPlaceholders.transformUp {
               case p if p == placeholder => childPlan
              }
            }    
          }
      }
    }
  }
  //过滤去掉不够高效的物理计划，但目前这里没有实现呢
  val pruned = prunePlans(plans)
  asset(pruned.hasNext, s "No plan for $plan")
  pruned
}
```
### 物理计划Strategy体系
参考SparkStrategies中各种策略的apply方法，作用是将传入的LogicalPlan转换为SparkPlan的列表。因此，Strategy是生成物理算子树的基础。  
在实现上，各种Strategy会匹配传入的LogicalPlan节点，根据节点或节点组合的不同情形实现一对一的映射或多对一的映射。一对一映射以BasicOperators为例，该Strategy实现了各种基本操作的转换，
其中列出了大量的映射关系，包括**Sort对应SortExec，Union对应UnionExec等**。多对一的情况涉及对多个LogicalPlan节点进行组合转换，这里称为逻辑算子树的模式匹配。   
目前，在Spark SQL中，逻辑算子树的节点模式有4种：
>* ExtractEquiJoinKeys
>* ExtractFiltersAndInnerJoins
>* PhysicalAggregation
>* PhysicalOperation: 匹配逻辑算子树中的Project和Filter等节点，返回投影列，过滤条件集合和子节点。

```text
GenericStrategy[PhysicalPlan <: TreeNode[PhysicalPlan]] <-- SparkStrategy(各种strategy)
```
### 常见Strategy分析
在SparkPlanner中默认添加了8种Strategy(2.1版本，2.3是10种)来生成物理计划，具体查看源码。  
理解之前案例中LogicalPlan到SparkPlan是如何转换的。**2.1和2.3的实现貌似略有不同，待考证**   

## 执行前的准备
物理计划的生成意味着用户的SQL语句已经成功转换为SparkPlan物理算子树。但是提交给Spark系统执行之前需要完成若干准备工作，对树形结构的物理计划进行全局的整合处理或优化，具体如下：
```text
lazy val executedPlan: SparkPlan = prepareForExecution(sparkPlan)
//处理过程基于若干规则，主要包括对Python中UDF的提取，子查询的计划生成等
protected def prepareForExecution(plan: SparkPlan): SparkPlan = {
  preparations.foldLeft(plan) { case (sp, rule) => rule.apply(sp) }
}
```   
物理执行计划的准备规则：
>* python.ExtractPythonUDFs  
>* PlanSubqueries: 特殊子查询物理计划处理
>* EnsureRequirements: 确保执行计划分区与排序正确性
>* CollapseCodegenStages: 代码生成相关，涉及到Tungshen，后续分析
>* ReuseExchange: Exchange节点重用
>* ReuseSubquery: 子查询重用

各个规则详细实现参考源码，重点是EnsureRequirements规则。     
