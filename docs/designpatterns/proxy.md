## 代理模式

##### 定义

代理模式为一个对象提供一个替身或占位符以控制对这个对象的访问。

##### 作用

用于控制对象的访问。

##### 结构

所谓代理，就是一个人或者机构代表另一个人或者机构采取行动。在一些情况下，一个客户不想或者不能够直接引用一个对象，而代理对象可以在客户端和目标对象之间起到中介作用。

##### 角色

- 抽象对象角色：生命力目标对象和代理对象的共同接口。

- 目标对象角色：定义了代理对象所代表的目标对象。

- 代理对象角色：代理对象内部含有目标对象的引用，从而可以在任何时候操作目标对象；代理对象提供一个与目标对象相同的接口，以便可以在任何时候替代目标对象。

##### 代码实现

~~~java
public interface Game{
    void operate(String compantName);
}
// 被代理类
public class BlizzardGame implements Game{
    public void operate(String compantName) {
        System.out.println("Game is operating by " + compantName + "!");
    }
}
// 代理类
public class NineCity implements Game{
    private Game game;
    public NineCity(Game game) {
        this.game = game;
    }
    public void operate(String companyName) {
        System.out.println("Localizate...");
        game.operate(companyName);
        System.out.println("Online activities...");
    }
}
@Test
public class MainClass {
    public static void main(String[] args) {
        Game proxy = new NineCity(new BlizzardGame());
        proxy.operate("");
    }
}
~~~

