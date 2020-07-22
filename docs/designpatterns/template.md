## 模板模式

##### 定义

模板方法模式是在一个方法中定义一个算法的骨架，而降一些步骤延迟到子类当中。使得子类可以在不改变算法结构的情况下，重新定义算法中的某些步骤。

##### 结构

模板方法是所有模式中最为常见的几个模式之一，是基于继承的代码复用的基本技术。

模板方法模式需要开发抽象类和具体子类的设计师之间的协作。一个设计师负责给出一个算法的轮廓和骨架，另一些设计师负责给出这个算法的各个逻辑步骤。

##### 角色

- 抽象模板Abstract Template角色：定义并实现一个模板方法，方法使用一个或多个抽象操作。
- 具体模板Concrete Template角色：实现父类所定义的一个或多个抽象方法。

##### 代码实现

~~~java
public abstract class AbstractTemplate{
    public void templateMethod(){
        abstractMethod();
        hookMethod();
        concreateMethod();
    }
    // 抽象方法，由子类实现
    protected abstract void abstractMethod();
    // 空方法
    protected void hookMethod(){}
    // 已实现的方法
    private final void concreteMethod(){
        ...
    }
}
public class ConcreteTemplate extends AbstractTemplate{
    //基本方法的实现
    public void abstractMethod() {
        ...
    }
    // 重写钩子方法
    public void hookMethod() {
        ...
    }
}
~~~

##### 应用

HttpServlet类，模板方法service()，其中定义了doPost()、doGet()等基本方法交由子类实现。