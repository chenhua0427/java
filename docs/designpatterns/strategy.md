# 策略模式

OO几个原则：

- 封装变化
- 多用组合，少用继承
- 针对接口编程，不针对实现编程

##### 定义

策略模式定义了算法族，分别封装起来，让他们之间可以互相替换，此模式让算法的变化独立于使用算法的客户。

##### 结构

策略模式是对算法的封装，是吧使用算法的责任和算法本身分割开来，委派给不同的对象管理。策略模式通常把一个系列的算法包装到一系列的策略类里面，作为一个抽象策略类的子类。一句话，“准备一组算法，并将每个算法封装起来，使得他们可以互换”。

##### 角色

- 环境Context角色：是有一个Strategy的引用

- 抽象策略Strategy角色：这是一个抽象类，通常由一个接口或抽象类实现，给出所有具体策略类所需的接口。
- 具体策略ConcreteStrategy角色：包装了相关算法或行为

##### 代码实现

~~~java
public class Context{
    private Stategy stategy;
    // 构造函数，传入一个具体的策略对象
    public Context(Stategy stategy){this.stategy = stategy;}
    // 执行策略方法
    public void contextInterface(){this.stategy.strategyInterface();}
}
public interface Strategy{
    public void strategyInterface();
}
// 具体策略
public class ConcreteStrategyA implements Strategy{
    public void strategyInterface(){...} // 执行相关算法业务
}
public class ConcreteStrategyB implements Strategy{
    public void strategyInterface(){...} // 执行相关算法业务
}
~~~

##### 策略模式缺点

1. 客户端必须知道所有策略类，并自行决定使用哪个策略。
2. 由于策略模式把每个具体的策略实现都单独封装成为类，如果备选的策略很多的话，那么对象的数目就会很多。

