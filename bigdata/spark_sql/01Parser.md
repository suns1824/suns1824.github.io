DSL的实现：
>* 涉及语法和语义，定义DSL中具体的元素
>* 实现词法分析器（Lexer）和语法分析器（Parser），完成对DSL的解析，最终转换为底层逻辑来执行   

了解一下ANTLR语法生成工具：ANTLR使用自上而下的递归下降分析方法构建语法分析树，还支持基于监听器模式和访问者模式的树遍历器。
基于ANTLR4实现一个计算器：
>* 一个g4文件(词法文法混合文件)
>* 命令行或者maven生成相应代码
>* 使用Visitor模式实现解析

```java
public class MyCaculatorVistor extends CaculatorBaseVisitor<Object> {
    @Override
    public Object visitAddOrSubtract(CalculatorParser.AddOrSubtractContext ctx) {
        Object object0 = ctx.expr(0).accept(this);
        Object object1 = ctx.expr(1).accept(this);
        if ("+".equals(ctx.getChild(1).getText())) {
            return (Float)object0 + (Float)object1;
        } 
        //......
    }
    
    @Override
    public Object visitFloat(CaculatorParser.FloatContext ctx) {
        return Float.parseFloat(ctx.getText());
    }
}
```
```java
public class Driver {
    public static void main(String[] args){
      String query = "3.1*(6.3-4.51)";
      CaculatorLexer lexer = new CaculatorLexer(new ANTLRInputStream(query));
      CaculatorParser parser = new CaculatorParser(new CommonTokenStream(lexer));
      CaculatorVisitor visitor = new MyCaculatorVisitor();
      System.out.println(visitor.visit(parser.expr()));
    }
}
```
### SparkSqlParser之AstBuilder
```
ParseInterface<--AbstracSqlParser<--CatalystSqlParser & SparkSqlParser   
AbstractSqlParser中定义了返回AstBuilder的函数，AstBuilder继承了Antlr4生成的默认的SqlBaseBaseVisitor，用于生成SQL对应的AST，
SparkSqlAstBuilder继承AstBuilder，并在其基础上定义了一些DDL的访问操作，主要在SparkSqlParser中调用。
```
**重点**:理解常见SQL生成的AST结构。