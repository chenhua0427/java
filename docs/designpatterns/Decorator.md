## 装饰模式

##### 定义

动态的将责任附加到对象上。若要增加功能，装饰者提供了比继承更具有弹性的解决方案。

##### 作用

增强对象的行为和职责。

##### 结构

装饰模式以对客户透明的方式动态的给一个对象附加上更多的责任。换言之，客户端并不会觉得对象在装饰前后有什么不同。可以在不使用子类的情况下，将对象功能加以扩展。

##### 角色

- 抽象构件Compoment角色：给出一个抽象接口，以规范准备接收附加责任的对象。
- 具体构件ConcreteCompoment角色：定义一个将要接收附加责任的类。
- 装饰Decorator角色：持有一个构件对象的实例，并定义一个与抽象构建接口一致的接口。
- 具体装饰ConcreatDecorator角色：负责给构件对象增加附加责任。

##### 代码实现

~~~java
public interface Beverage{
    public double cost();
}
public class MilkTea implements Beverage{
    public double cost(){ retrun 5;}
}
public abstract class Decortator implements Beverage{
    protected Beverage beverage;
    public Decorator(Beverage bev){
        this.Beverage = bev;
    }
}
public class Coconut extends Decorator{
    public Coconnut(Beverage bev){
        super(bev);
    }
    public double cost(){
        return beverage.cost() + 4.3; //装饰
    }
}
@Test
public static void main(String[] args){
    Beverage beverage = new MilkTea();
    Decorator decorator = new Coconut(beverage);// 1.通过传入不同的装饰者，实现差异
    System.out.println(decorator.cost()); // 2.但是调用方式始终不变
}
~~~

