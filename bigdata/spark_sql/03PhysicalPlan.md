**物理计划时Spark SQL整个查询处理流程的最后一步，与底层平台紧密相关。在此阶段，Spark SQL会对生成的逻辑算子树进一步处理，得到物理算子树，并将LogicalPlan节点及其所包含的各种信息映射成Spark Core计算模型的元素，
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
