**在此阶段，字符串形态的SQL语句转换为树结构形态的逻辑算子树，SQL中包含的各种处理逻辑(过滤，剪裁等)和数据信息都会被整合在逻辑算子树的不同节点中。本质上，逻辑计划是一种中间过程表示，
与Spark平台无关。**
## 概述
```text
SQL Query --- (SparkSqlParser.parse) --- Unresolved LogicalPlan --- (Analyzer) --- Analyzed LogicalPlan --- (Optimizer) --- Optimized LogicalPlan
```
分为3个阶段：
>* SparkSqlParser中的AstBuilder执行节点访问，将语法树的各种Context节点转换为对应的LogicalPlan节点，从而成为一棵未解析的逻辑算子树(不包含数据信息和列信息)。
>* Analyzer将一系列的规则作用在Unresolved LogicalPlan，对树上的节点绑定各种数据信息，生成解析后的逻辑算子树。
>* 由Spark SQL中的优化器(Optimizer)将一系列优化规则作用到上一步的Analyzed LogicalPlan，确保结果正确的前提下改写其中的低效结构，生成优化后的逻辑算子树。  

## LogicalPlan简介
### QueryPlan
QueryPlan的主要操作分为6个模块：**输入输出，字符串，规范化，表达式操作，基本属性，约束**
### LogicalPlan
```text
//变量
resolved
canonicalized
//方法
解析状态相关
节点基本信息
如何解析相关
```
LogicalPlan仍是抽象类，根据子节点数目，绝大部分的LogicalPlan可以分为3类：
>* LeafNode,该类型的LogicalPlan节点对应数据表和命令相关的逻辑
>* UnaryNode， 数据的逻辑转换操作
>* BinaryNode，包括join和集合操作以及CoGroup  

## AstBuilder机制：UnResolved LogicalPlan生成
如何从SQL抽象语法树生成相应的逻辑算子树：  
**Spark SQL在ParserDriver中调用语法分析器的singleStatement()方法构建整个语法树，然后通过AstBuilder访问者类对语法树访问：**   
```text
//以之前的spark.sql("select name from student where age > 18")为例
override def visitSingleStatement(ctx: SingleStatementContext): LogicalPlan = withOrigin(ctx) {
    visit(ctx.statement).asInstanceOf[LogicalPlan]
    //对根节点的访问操作会递归访问其子节点(ctx.statement),逐步向下递归调用，直到访问某个子节点时能够构造LogicalPlan，然后传递给父节点，因此返回的结果可以转换为LogicalPlan类型
}

//注册context的origin，任何在closure中TreeNode都要被分配一个注册的orgin
def withOrigin[T](ctx: ParserRuleContext)(f: => T): T = {
    val current = CurrentOrigin.get
    CurrentOrigin.set(position(ctx.getStart))
    try {
      f
    } finally {
      CurrentOrigin.set(current)
    }
 }

Catalyst提供了节点位置功能，即能够根据TreeNode定位到对应的SQL字符串中的行数和起始位置----Origin提供了line和 startPosition两个构造参数，
分别代表行号和偏移量。在CurrentOrigin对象中，提供了各种set和get操作还有withOrigin方法。                                         
  
```
**ps:需要结合抽象语法树中各个Context的节点关系思考**:
```text
SingleStatementContext ---...--- QuerySpecificationContext --- FromClauseContext  
```
在访问QuerySpecificationContext节点时，执行逻辑可以看作两部分，如下代码：首先访问FromClauseContext子树，生成名为from的LogicalPlan，接着在from上继续扩展。
```text
override def visitQuerySpecification(ctx: QuerySpecificationContext): LogicalPlan = withOrigin(ctx) {
  val from = OneRowRelation.optional(ctx.fromClause) {
    visitFromClause(ctx.fromClause)
  }
  withQuerySpecificaiton(ctx, from)
}
```
总的来说，生成UnresolvedLogicalPlan的过程从访问QuerySpecificaitonContext开始，分为3步：
>* 生成数据表对应的LogicalPlan，访问FromClauseContext并递归访问，直到匹配到TableNameContext节点(visitTableName)，直接根据TableNameContext中的数据信息生成UnResolvedRelation，
此时不再递归访问子节点，构造名为from的LogicalPlan并返回。
>* 生成加入了过滤逻辑的LogicalPlan，在QuerySpecificationContext包含了名称为where的BooleanExpressionContext类型(BooleanDefaultContext节点)。AstBuilder会对会对该子树进行递归
访问，这里遇到ComparisonContext节点会生成GreaterThan表达式，生成expression并返回作为过滤条件，然后基于此过滤条件生成Filter LogicalPlan节点。此LogicalPlan和第一步中的UnRsolvedRelation
构成名为withFilter的LogicalPlan，其中Filter节点为根节点。
>* 生成加入列剪裁后的LogicalPlan：这里对应‘select name’。AstBuilder在访问过程中会获取QuerySpecificationContext节点所包含的NamedExpressionSeqContext成员，并对其所有子节点对应
的表达式进行转换，生成NameExprssion列表，然后基于此生成Project LogicalPlan。最后，由此LogicalPlan和第二步中的withFilter构造名称为withFilter的LogicalPlan，其中Project最终会成为
整棵逻辑算子树的根节点。

**重点：理解逻辑算子树中各节点的Expression生成！** 

## Analyzer机制：Analyzed LogicalPlan生成
Analyzer起到的作用就是将Unresolved LogicalPlan中未解析的UnresolvedRelation和UnresolvedAttribute两种对象解析成有类型的对象。这就需要Catalog的相关信息。
### Catalog体系
在关系型数据库中Catalog通常可以理解为一个容器或数据库对象命名空间中的一个层次，主要用来解决命名冲突等问题。  
在Spark SQL系统中，Catalog主要用于个各种函数资源信息和原数据信息(数据库/表/视图/分区/函数)的统一管理。  
Catalog体系实现以SessionCatalog为主体，通过SparkSession提供给外部调用。SessionCatalog封装了元数据信息和函数信息等，构造参数包括6部分：
>* 传入Spark SQL信息的CatalystConf
>* 传入Hadoop配置信息的Configuration
>* 全局的临时视图管理(GlobalTempViewManager，提供了对全局视图的原子操作，包括创建，更新，删除，重命名等，主要功能依赖一个mutable类型的HashMap对视图名和数据原进行映射，其中value是视图
所对应的LogicalPlan)
>* FunctionResourceLoader(函数资源加载器)，加载自定义函数和Hive中的各种函数
>* FunctionRegistry(函数注册接口)，用来实现对函数的注册，查找和删除等功能，线程安全。
>* ExternalCatalog，用来管理数据库/表/分区和函数的接口，目标是与外部系统交互并做到上述内容的非临时性存储，线程安全。  

总体来看，SessionCatalog是管理上述一切信息的入口，除上述的构造函数外，内部还包括一个mutable类型的HashMap来管理临时表信息，以及currentDb变量来指代当前操作所对应的数据库名称。    
### Rule体系
在Unresolved LogicalPlan逻辑算子树的操作(绑定，解析，优化等),主要方法都是基于规则的，通过scala模式匹配规则进行树结构的转换或者节点改写。
```text
abstract class Rule[TreeType <: TreeNode[_]] extends Logging {
  val ruleName: String = {
    val className = getClass.getName
    if (className endsWith "$") className.dropRight(1) else className
  }
  def apply(plan: TreeType): TreeType
}
```
有不同的Rule实现，还需要驱动程序来调用这些规则：**RuleExecutor**。   
RuleExecutor内部提供了一个Seq[Batch]。Batch代表一套规则，配备一个策略。RuleExecutor的apply(plan: TreeType): TreeType方法会按照batches顺序和batch内的Rules顺序，对传入的plan里的节点进行迭代处理，
处理逻辑由具体Rule子类实现：
```text
def execute(plan: TreeType): TreeType = {
  var curPlan = plan
  batches.foreach { batch => 
    val batchStartPlan = curPlan
    var iteration = 1
    var lastPlan = curPlan
    var continue = true
    while (continue) {
     curPlan = batch.rules.foldLeft(curPlan) {
       case (plan, rule) => rule(plan)
     }
     iteration += 1
     if (iteration > batch.strategy.maxIterations) {
       continue = false
     }
     if (curPlan.fastEquals(lastPlan)) {
       continue = false
     }
     lastPlan = curPlan*
    }
  }
}
```
### Analyzed LogicalPlan生成过程
因为继承自RuleExecutor类，所以Analyzer执行过程中会调用其父类RuleExecutor中实现的run方法，Analyzer中重新定义了一系列规则**baches**。2.1版本中Analyzer默认定义了6个Bach，外加额外的扩展规则一共34条。
具体查阅资料或查看源码，举几个例子，看不懂查资料：
>* CTESubstitution: 对应With语句，在SQL中主要用于子查询模块化，作用：将多个LogicalPlan合并为一个LogicalPlan
>* EliminiateUnions: 在Union算子节点只有一个子节点时，union操作没有起到作用，需要消除Union节点，作用：将Union(children)替换为children.head节点
>* ......

在QueryExecution中：
```text
//该方法会循环调用规则对逻辑算子树分析
val analyzed: LogicalPlan = analyzer.execute(logical)
```
以之前例子来看，在Unresolved LogicalPlan中，Analyzer中首先匹配的是ResolveRelations规则(解析数据表)：
```text
object ResolveRelations extends Rule[LogicalPlan] {
  //得到分析后的LogicalPlan，该例中数据表中每个字段类型都得到分析
  private def lookupTableFromCatalog(u: UnresolvedRelation): LogicalPlan = {
    try {
      catalog.lookupRelation(u.tableIdentifier, u.alias)
    } catch {
      ...
    }
  }
  def apply(plan: LogicalPlan): LogicalPlan = plan resolveOperators {
    case i @ InsertIntoTable(u: UnresolvedRelation, parts, child, _, _) if child.resolved => 
      i.copy(table = EliminateSubqueryAliases(lookuptableFromCatalog(u)))
    case u: UnresolvedRelation =>
      val table = u.tableIndentifier
      if (table.database.idDenfined && conf.runSQLonFile && !catalog.isTemporaryTable(table) && (!catalog.databaseExists(table.database.get) || !catalog.tableExists(table))) {
        u
      } else {
        lookupTableFromCatalog(u)
      }
  }
}
```
接下来执行第二步，执行ResolveReferences规则，主要是Filter节点中age的信息从Unresolved状态变为Analyzed状态(表示Unresolved状态的前缀字符单引号被去掉)；  
然后会调用TypeCoercion规则集中的ImplicitTypeCasts规则，对表达式中的数据类型进行隐式转换，此时FIlter节点变为Analyzed状态；    
最后执行ResolveReferences规则，Project节点完成解析。   

## Spark SQL优化器Optimizer
Analyzed LogicalPLan在本例中自底向上节点分别为Relation，Subquery,Filter,Project算子。  
Unresolved LogicalPlan --> Analyzed LogicalPlan是一一转换而来的。但是有可能有低效的写法，所以需要针对逻辑算子树的优化器**Optimizer**。  
### Optimizer概述 
Optimizer同样继承自RuleExecutor类，在QueryExecution中，Optimizer会对传入的Analyzed LogicalPlan执行execute()方法：
 ```text
val optimizedPlan: LogicalPlan = optimizer.execute(analyzed)
```
与Analyzer类似，Optimizer也依赖一些列的规则,有16个batch和53个规则(2.1版本)，不举例说明了。
### Optimized LogicalPlan的生成过程
 对于案例生成的Analyzed LogicalPlan，   
 首先执行的是Finish Analysis这个Batch中的Eliminate-SubqueryAlias优化规则，用来消除子查询别名的情形。
 ```text
object EliminateSubqueryAliases extends Rule[LogicalPlan] {
  def apply(plan: LogicalPlan): LogicalPlan = plan transformUp {
  //直接作用于子节点
    case SubqueryAlias(_, child, _) => child
  }
}
```
所以，SubqueryAlias节点被删除，Filter节点直接作用于Relation节点。 
接下来将匹配Operator Optimizations这个Batch中的InferFiltersFromConstraints优化规则，用来增加过滤条件(不会与当前算子或其子节点现有的过滤条件重叠)，简化版代码：
```text
object InferFiltersFromContraints extends Rule[LogicalPlan] with PredicateHelper {
  def apply(plan: LogicalPlan): LogicalPlan = plan transform {
    case filter @ Filter(condition, child) =>
          val newFilters = filter.constraints --
            (child.constraints ++ splitConjunctivePredicates(condition))
            //当新的过滤条件不为空时，会与现有的过滤条件进行整合，构造新的Filter逻辑算子节点
          if (newFilters.nonEmpty) {
            Filter(And(newFilters.reduce(And), condition), child)
          } else {
            filter
          }
  }
}
```
本例中，Filter逻辑算子树节点中多路“isnotNull(age#0L)”这个过滤条件。   
最后一步，上述逻辑算子树会匹配Operator Optimizations这个Batch中的ConstantFolding优化规则，对LogicalPlan中可以折叠的表达式进行静态计算直接得到结果，简化表达式：
```text
object ConsantFoldingFolding extends Rule[LogicalPlan] {
  def apply(plan: LogicalPlan): LogicalPlan = plan transform {
    case q: LogicalPlan => q transfromExpressionsDown {
     case 1: Literal => 1
     //如果LogicalPlan中的表达式可以折叠，会将EmptyRow作为参数传递到其eval方法中直接计算，然后根据计算结果构造Literal常量表达式。
     case e if e.foldable => Literal.create(e.eval(EmptyRow), e.dataType)
    }
  }
}
```
以上就是逻辑算子树的生成分析和优化阶段。  