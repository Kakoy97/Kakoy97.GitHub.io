---
layout:     post
title:      "「装饰者模式」咖啡店"
subtitle:   "Decorator"
date:       2021-10-28 23:04:00
author:     "Kakoy"
header-img: "img/2021-bg.jpg"
catalog: true
tags:
    - 设计模式
---

> “ 装饰者模式 ”

## 问题
咖啡店需要设计一个订单系统，购买咖啡时，要求可以在其中加入各种调料,比如牛奶,豆浆,摩卡等,咖啡店会根据所加入的调料收取不同的费用。
原先父类的设计如下面所示
```cs
public class Beverage
{
    string description;
    
    public string GetDescription(){
        return description;
    }

    public abstract double Cost();
}
```
如果单单将每种咖啡与调料组合成一个子类继承于Beverage,那么类可以成千上万,这种**类爆炸**绝对是不合理的。

使用装饰者模式可以很好的解决这个问题。首先让我们看看装饰者模式的说明:<br>
**装饰者模式动态的将责任附加到对象上，诺要扩展功能，装饰者提供了比继承更有弹性的替代方案。**<br>
1.装饰者和被装饰对象有相同的超类型。<br>
2.你可以用一个或多个装饰者包装一个对象。<br>
3.既然装饰者和被装饰者对象有相同的超类型,所以在任何需要原始对象(被包装的)场合,可以用装饰过的对象代替它。<br>
4.装饰者可以在所委托被装饰者的行为之前或之后，加上自己的行为，以达到特定的目的。<br>
5.对象可以在任何时候被装饰，所以可以在运行时动态的不限量的用你喜欢的装饰者来装饰对象。 <br>
    
看看我们的框架图
![img](/img/headFirst/Decorator01.jpg)

但是我一直很不理解,为什么需要有一个CondimentDecorator来派生出调料呢,不如直接让调料和咖啡一样,直接继承于超类。

## 代码

```cs
public abstract class Beverage //超类
{
    public string discription = "unknow Name";

    public virtual string GetDiscription(){
        return discription;
    }
    public abstract double Cost();
}
```
在写C#的时候遇到了一个坑,如果父类的方法没有 Virtual 关键字, 子类是无法重写父类的方法的。
导致我最后输出咖啡名字的时候,一直显示的是"unknow Name"

```cs
public class DarkRoast : Beverage //被装饰者咖啡
{
    public DarkRoast(){
        discription = "DarkRoast";
    }

    override public double Cost(){
        return 1.5;
    }
}
```

```cs
public class Mike : Beverage //装饰者调料
{
    Beverage beverage;
    public Mike(Beverage beverage)
    {
        this.beverage = beverage;
    }
    public override double Cost()
    {
        return beverage.Cost() + 0.5;
    }

    public override string GetDiscription()
    {
        return beverage.GetDiscription() + " Mike";
    }
}

```
我直接让调料继承于超类，省略了CondimentDecorator,可能是C#与java语法的不同,导致如果我CondimentDecorator也是抽象类的话,
我的装饰者调料重写GetDiscription()方法失效,最终还是调到超类的GetDiscription(),打印出的是"unknow Name"

附上我写的CondimentDecorator,虽然项目里没有运用到,但是希望大神们能替我解答一下。
```cs
public abstract class CondimentDecorator : Beverage
{
    public abstract string GetDiscription();
}
```

最后看看我们的main方法和执行结果
```cs
public class Main3 : MonoBehaviour 
{
    void Start()
    {
        Beverage darkRoast = new DarkRoast(); //这次我只用了一个来装饰咖啡
        darkRoast = new Mike(darkRoast);
        //darkRoast = new MoCha(darkRoast); //可以无限包装
        //...
        print("价格:" + darkRoast.Cost());
        print("名字:" + darkRoast.GetDiscription());
    }
}
```
最终结果:
![img](/img/headFirst/Decorator02.jpg)

好了,今天的装饰者模式也让我学到了一个新的设计原则:**类应该对扩展开放,对修改关闭**
还有许多新的要点:<br>
    1.继承属于扩展形式之一，但不见得是达到弹性设计的最佳方式。<br>
    2.组合和委托可用于在运动时动态的加上新的行为。<br>
    3.除了继承，装饰者模式也可以让我们扩展行为。<br>
    4.装饰者模式意味着一群装饰者类，这些类用来包装具体条件。<br>
    5.装饰者一般对组件的客户是透明的，除非客户程序依赖于组件的具体类型。<br>
    6.装饰者会导致设计中出现许多小对象，如果过度使用，会让程序变得复杂。<br>