调整点： 
1. Spark SQL在物理计划生成方面还有很多工作要做，例如，对生成的物理计划列表进行过滤筛选(prunePlans)在当前版本并没有实现，生成多个物理计划后，仅仅是直接选取列表
中的第一个作为结果。参考QueryPlanner中的plan方法并没有实现prunePlans方法。
调用处：
 ```text
//QueryEcecution
lazy val sparkPlan: SparkPlan = {
    SparkSession.setActiveSession(sparkSession)
    // TODO: We use next(), i.e. take the first plan returned by the planner, here for now,but we will implement to choose the best plan.
    planner.plan(ReturnAnswer(optimizedPlan)).next()
  }
```

2. 2.1中ExchangeCoordinator功能相对简单，仅用于确定Shuffle后的分区数目。  


不解之处：  
1. 同一个窗口可能对应多个窗口表达式？  
