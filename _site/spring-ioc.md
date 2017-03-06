# Spring IOC

### 装配Bean

任何一个成功的应用都是由多个为了实现某一个业务目标而互相协作的组建构成的,这些组建必须互相了解,并互相协作来完成工作. 在Spring中,对象无需自己负责查找或创建与其关联的其他对象,容器负责把需要互相协作的对象引用赋予各个对象.创建对象之间协作关系的行为就称为**装配**.

Spring容器提供两种配置Bean的方式: 

1. 基于XML文件作为配置文件
2. 基于注解

基于XML文件的配置是比较常见的一种配置方法,在`<beans>`标签内,放置所有Spring的配置信息,包括`<bean>`元素的声明.`<bean>`将会在Spring容器中创建一个对象,他的id是在Spring中的命名,唯一,可以创建一个class属性对应的类的对象实例.

**构造器注入**

```
<bean id='id' class="className">
    <constructor-arg value='constructor-param-value'/>
</bean>
```
这个例子中我们可以看到可以使用构造器注入bean, 这个时候的参数是基本类型.那能否通过构造器注入一个对象引用呢? sure!

```
<bean id='id' class="className">
    <constructor-arg value='1'/>
    <constructor-arg ref="another-bean-id"/>
</bean>
```
这样就往构造器中注入了一个id为"another-bean-id"的bean的对象引用了.

**单例模式** 

我们还可以通过**工厂方法**创建Bean.下面我们来写一个[单例类](http://wenrisheng.iteye.com/blog/1996738)

```
public class Stage {
    private Stage(){}
    private static class StageSingletonHolder() {
        static Stage instance = new Stage();
    }
    public static Stage getInstance() {
        return StaeSingletonHolder.instance;
    }
}
```
单例模式中,构造器方法将被私有化防止被实例化,静态方法getInstance()每次调用都会返回同一个实力(注意: 这里是线程不安全的).在这个单例类中并没有公开的构造方法,我们可以通过`<bean>`元素的factory-method属性来指定一个静态方法来创建一个类的实例.
```
<bean id="stage" class="Stage" factory-method='getInstance'/>
```
如果要装配的对象需要通过静态方法来创建实例,那么factory-method就可以被适用.

* * *

### Bean的作用域

在Spring中,Bean默认都是单例的如果我们希望每次每次容器分配一个Bean时都返回一个新的实例,那要如何覆盖默认的Spring配置呢?

| 作用域 | 说明 |
| ------| ------ | ------ |
| singleton | 默认, 一个Bean定义只有一个对象实例|
| prototype | 每次调用都创建一个实例|
| request | 在一次HTTP请求中,每个Bean定义对应一个实例, 该作用域仅基于Web的Spring上下文中有效(Spring MVC) |
| sessoin | 在一个HTTP Session中,每个Bean定义对应一个实例, 该作用域仅基于Web的Spring上下文中有效(Spring MVC)|
| global-session| 全局的HTTP session中, 每个Bean定义对应一个实例|

```
<bean id="ticket" class="Ticket" scope="prototype">
```
这样,每次都会实例化出一个新的Ticket的对象实例.

### Bean的初始化与销毁

当实例化一个Bean的时候,可能需要执行一些初始化操作来确保Bean可用,比如做一些数据的准备初始化工作,同理在销毁一个Bean的时候,也可能需要做一些额外的销毁工作. `init-method`指定初始化Bean的方法, `destroy-method`指定了销毁Bean的方法.

* * *

### 注入Bean属性
在一个Class中往往会有很多属性, 这些属性也是可以通过Spring注入进去的.使用`<property>`来配置Bean的属性.

```
public class Singer {
    private String name;
    private Instrument instrument;
    public void setName(String Name)
        this.name = name;
    public String getName()
        return this.name;
    public void setInstrument(Instrument instrument)
        this.instrument = instrument;
    public Instrument getInstrument()
        return this.instrument;
}
```
这里我们看到有一个name属性,我们可以通过下面的方式来装配.
```
<bean id='singer' class="Singer" >
    <property name="name" value="allen">
    <property name="instrument" ref="instrument">
</bean>
```
我们可以看到就像构造器装配Bean一样, 属性的装配一样很方便.除此之外我们还可以装配集合哟.`<List>`, `<set>`, `<map>`, `<props>`. 这里就不详细介绍了,比较简单,查查文档吧.


