[Spark SQL基础教程](https://www.jianshu.com/p/e02bc3da179b)  
```text
val spark = new SparkSession.builder().appName("sql_example").master("local").getOrCreate()
spark.read.json("student.json").createOrReplaceTempView("student")  //创建数据表并读取数据
spark.sql("select name from student where age > 18").show()
```
一般来讲，对于Spark SQL系统，从SQL到Spark中RDD的执行需要经过两个大的阶段，分别是逻辑计划和物理计划：
```text
SQL Query ----> Unresolved LogicalPlan ----> Analyzed LogicalPlan ----> Optimized LogicalPlan --
--> Iterator ----> SparkPlan ----> PreparedSparkPlan ----> show()  
``` 
未解析的逻辑算子树，仅仅是数据结构，不包含任何数据信息，解析后的逻辑算子树，节点绑定各种信息，和优化后的逻辑算子树(应用各种优化规则对一些低效的逻辑计划进行转换)，
根据逻辑算子树，生成物理算子树的列表Iterator，，从列表中按照一定的策略选出最优的物理算子树(SparkPlan)，最后，对选取的物理算子树进行提交前的准备工作。  

Catalyst：
InternalRow用来表示一行行数据的类；  
TreeNode是Spark SQL中所有树结构的基类；   
表达式一般指的是不需要触发执行引擎而能够直接进行计算的单元，如果说TreeNode是框架，Expression是灵魂。
Expression类定义了5个方面的操作，包括**基本属性，核心操作(eval函数实现了表达式对应的处理逻辑，genCode等)，输入输出，字符串表示(用于查看该Expression的具体内容)，等价性判断**。   
基本属性和操作：
>* foldable: 标记表达式是否能在查询执行之前直接静态计算
>* deterministic: 标记表达式是否为确定性的，即每次eval函数输出是否相同
>* nullable: 标记表达式是否可能输出null值
>* references: 返回值为AttributeSet值，表示该Expression中会涉及的属性值
>* canonicalized: 返回经过规范化处理(Canonicalize类)后的表达式
>* semanticEquals: 判断两个表达式在语义上是否等价  

**重点**理解TreeNode体系！  