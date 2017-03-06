# Spring 初体验

说到Spring我们就会想为什么项目中要用到Spring框架呢? Spring的使命就是为了**简化Java开发**.为了降低Java开发的复杂性, Spring主要采取了下面几个关键策略:

* 基于POJO的轻量级和最小入侵性编程
* 通过依赖注入和面向接口实现松耦合
* 基于切面和惯例进行声明式编程
* 通过切面和模板减少样板式代码

### 关于依赖注入

实际应用中一个类往往由多个类组成, 这些类互相之间进行协作完成实现特定的业务逻辑,每个对象负责管理与自己相互协作的对象(即他所依赖的对象)的引用,,这将会导致高度耦合和难以测试.

例如:

```
public class RecuingKnight implements Knight {
    private OtherClass otherClass;
    public RecuingKnight() {
        otherClass = new OtherClass();
    }
    public void test() {
        otherClass.method();
    }
}
```

从例子中我们可以看到RecuingKnight的构造方法自行创建了OtherClass对象的引用.这就使得RecuingKnight紧密的与OtherClass耦合在一起.为了编写RecuingKnight的测试代码,必须保证test()方法被调用的时候otherClass的method()方法也被调用,但是没有一个简单的方式能够实现这一点,这样RecuingKnight将无法进行测试.

通过依赖注入, 对象的依赖关系将有负责协调系统中各个对象的第三方组件在创建对象时设定.对象无需自行创建或管理他们的依赖关系--依赖关系将被自动注入到需要它们的对象中去.

```
public class RecuingKnight implements Knight {
    private OtherClass otherClass;
    public RecuingKnight(OtherClass otherClass) {
        this.otherClass = otherClass;
    }
    public void test() {
        otherClass.method();
    }
}
```

上面这种方式就是构造器注入, 这种方式RecuingKnight将不需要自行创建OtherClass对象,而是通过构造器的参数传入.
