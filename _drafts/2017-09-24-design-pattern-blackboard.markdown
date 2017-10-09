---
layout: post
title: 设计模式黑板报
date: 2017-09-24 10:00:00 +0800
tags:
- DesignPattern
---

高考前教室的后黑板有时会被用来划重点，笔者将本文类比为黑板报，就是为了能够不时回过头来看看这些设计模式，以加深印象。本文包含 [《Head First Design Patterns》](http://shop.oreilly.com/product/9780596007126.do) 一书中提到的所有设计模式。

<h4>🔥 策略模式(The Strategy Pattern)</h4>

![The Strategy Pattern](/assets/201709/strategy_pattern.png)

策略模式定义了一组算法，每个算法都用一个类封装，并且之间可以相互替换。策略模式使得算法独立于使用它的客户端。

<h4>🔥 观察者模式(The Observer Pattern)</h4>

![The Observer Pattern](/assets/201709/observer_pattern.png)

**接口**
```java
public interface Subject {
  void registerObserver(Observer o);
  void removeObserver(Observer o);
  void notifyObservers();
}

public interface Observer {
  void update(float temp, float humidity, float pressure);
}

public interface DisplayElement {
  void display();
}
```

**WeatherData 实现 Subject 接口**
```java
public class WeatherData implements Subject {
  private List<Observer> observers = new ArraryList<>();
  private float temperature;
  private float humidity;
  private float pressure;

  @Override
  public void registerObserver(Observer o) {
    observers.add(o);
  }

  @Override
  public void removeObserver(Observer o) {
    int i = observers.indexOf(o);
    if (i >= 0) {
      observers.remove(o);
    }
  }

  @Override
  public void notifyObservers() {
    for (Observer o: observers) {
      o.update(temperature, humidity, pressure);
    }
  }

  public void measurementsChanged() {
    notifyObservers();
  }

  public void setMeasurements(float temperature, float humidity, float pressure) {
    this.temperature = temperature;
    this.humidity = humidity;
    this.pressure = pressure;
    measurementsChanged();
  }

  // 其他方法略
}
```

**CurrentConditionsDisplay  其中一种Display的实现类**
```java
public class CurrentConditionsDisplay implements Observer, DisplayElement {
  private float temperature;
  private float humidity;
  private Subject weatherData;  // 这里是重点

  public CurrentConditionsDisplay(Subject weatherData) {
    this.weatherData = weatherData;
    weatherData.registerObserver(this); // 将自己注册到weatherData的列表中
  }

  public void update(float temperature, float humidity, float pressure) {
    this.temperature = temperature;
    this.humidity = humidity;
    display();
  }

  public void display() {
    System.out.println("Current conditions: " + temperature +
        "F degrees and " + humidity + "% humidity");
  }
}
```

**测试代码**
```java
public class WeatherStation {
  public static void main(String[] args) {
    WeatherData weatherData = new WeatherData();
    CurrentConditionsDisplay currentDisplay = new CurrentConditionsDisplay(weatherData);
    StatisticsDisplay statisticsDisplay = new StatisticsDisplay(weatherData);
    ForecastDisplay forecastDisplay = new ForecastDisplay(weatherData);

    weatherData.setMeasurements(80, 75, 30.4f);
    weatherData.setMeasurements(82, 70, 29.2f);
    weatherData.setMeasurements(78, 90, 29.2f);
  }
}
```

**Java's build-in Observer Pattern**

![Java's build-in Observer Pattern](/assets/201709/java_buildin_observer_pattern.png)

**使用java.util.Observable重写WeatherData**
```java
import java.util.Observable;
import java.util.Observer;

public class WeatherData extends Observable { // Observable 是类
  private float temperature;
  private float humidity;
  private float pressure;

  public void measurementsChanged() {
    setChanged();       // 在notifyObservers前调用setChanged()指明状态发生变化
    notifyObservers();  // 使用未带参数的notifyObservers()意味着在Observer端进行拉数据（pull model）
  }

  public void setMeasurements(float temperature, float humidity, float pressure) {
    this.temperature = temperature;
    this.humidity = humidity;
    this.pressure = pressure;
    measurementsChanged();
  }

  // 以下三个方法在之前的写法中略去了，因为现在我们要使用"pull"模式，观察者需要使用这些方法
  public float getTemperature() {
    return temperature;
  }

  public float getHumidity() {
    return humidity;
  }

  public float getPressure() {
    return pressure;
  }
}
```

**使用java.util.Observer重写CurrentConditionsDisplay**
```java
import java.util.Observable;
import java.util.Observer;

// 现在我们实现的是 java.util 中的 Observer, 它依然是一个接口
public class CurrentConditionsDisplay implements Observer, DisplayElement {
  Observable observable;
  private float temperature;
  private float humidity;

  public CurrentConditionsDisplay(Observable observable) {
    this.observable = observable;
    observable.addObserver(this);
  }

  public void update(Observable obs, Object args) {
    if (obs instanceof WeatherData) {
      WeatherData weatherData = (WeatherData)obs;
      this.temperature = weatherData.getTemperature();  // 拉取数据
      this.humidity = weatherData.getHumidity();
      display();
    }
  }

  public void display() {
    System.out.println("Current conditions: " + temperature +
        "F degrees and " + humidity + "% humidity");
  }
}
```

观察者模式定义了对象之间**一对多**的依赖关系，当**一**对应的对象状态有所变化，所有依赖它的对象都会被通知并自动更新。

<h4>🔥 装饰者模式(The Decorator Pattern)</h4>

装饰者模式的基本类图结构如下：

![Common Decorator Pattern](/assets/201709/common_decorator_pattern.png)

Starbuzz饮品装饰者类图结构如下：

![Beverage Decorator Pattern](/assets/201709/beverage_decorator_pattern.png)

**重点：** Decorator 使用继承是为了获取类型匹配（type matching），而不是行为（behavior）.

**Beverage类不需变动**
```java
public abstract class Beverage {
  String description = "Unknown Beverage";
  public String getDescription() {
    return description;
  }

  public abstract double cost();
}
```

**调味品（Decorator）的抽象类**
```java
// 调味品可以跟饮品相互替换（Interchangeable），所以继承Beverage
public abstract class CondimentDecorator extends Beverage {
  // 所有的调味品都必须重新实现getDescription()方法
  public abstract String getDescription();
}
```

**两种饮品的实现**
```java
public class Espresso extends Beverage {  // 浓咖啡
  public Espresso() {
    description = "Espresso";
  }

  public double cost() {
    return 1.99;
  }
}

public class HouseBlend extends Beverage { // 混合咖啡
  public HouseBlend() {
    description = "House Blend Coffee";
  }

  public double cost() {
    return .89;
  }
}
```

**一种调味品（Decorator）的实现**
```java
public class Mocha extends CondimentDecorator { // 摩卡是一个装饰者，所以继承 CondimentDecorator
  Beverage beverage;    // 对被装饰的类进行包装

  public Mocha(Beverage beverage) { // 构造装饰者的时候使用被装饰的类
    this.beverage = beverage;
  }

  public String getDescription() {
    return beverage.getDescription + ", Mocha";
  }

  public double cost() {
    return beverage.cost() + .20;
  }
}
```

**测试代码**
```java
public class StarbuzzCoffee {
  public static void main(String args[]) {
    // 点一杯浓咖啡，不要任何调味品
    Beverage beverage = new Espresso();
    System.out.println(beverage.getDescription() + "$" + beverage.cost());

    // 点一杯深焙咖啡，加双份摩卡和一份奶油
    Beverage beverage2 = new DarkRoast();
    beverage2 = new Mocha(beverage2);   // 第一份摩卡
    beverage2 = new Mocha(beverage2);   // 第二份摩卡
    beverage2 = new Whip(beverage2);    // 一份奶油
    System.out.println(beverage2.getDescription() + "$" + beverage2.cost());
  }
}
```

java.io包就是一个装饰者模式的一个实例：

![java.io package](/assets/201709/java_io_package.png)

![java.io package 2](/assets/201709/java_io_package2.png)

装饰者模式动态地附加一个对象的责任。装饰器提供了用于扩展功能的子类的灵活替换。

<h4>🔥 工厂模式(The Factory Pattern)</h4>

工厂模式分为简单工厂模式（The Simple Factory）、工厂方法模式（The Factory Method Pattern）和抽象工厂模式（The Abstract Factory Pattern）。

简单工厂模式其实算不上是一种模式，它更像是一种编程习惯（programming idiom）。本文不做介绍。

工厂方法模式（The Factory Method Pattern）定义: 工厂方法模式定义了一个创建对象的接口，并让子类决定实例化对象的类型。工厂方法将实例化推迟得到子类。

![The Factory Method Pattern](/assets/201709/factory_method_pattern.png)

以 PizzaStore 为例:

![The Pizza Factory Method Pattern](/assets/201709/pizza_factory_method_pattern.png)

换个视角来看 PizzaStore 类图:

![The Pizza Factory Method Pattern](/assets/201709/pizza_factory_method_pattern2.png)

抽象工厂模式（The Abstract Factory Pattern）定义: 抽象工厂模式可以向客户端提供一个接口，使客户端在不必指定具体类的情况下，创建多个相关或独立的对象。

![The Abstract Factory Pattern](/assets/201709/abstract_factory_pattern.png)

以 PizzaStore 为例:

![The Pizza Abstract Factory Pattern](/assets/201709/pizza_abstract_factory_pattern.png)

<h4>🔥 单例模式(The Singleton Pattern)</h4>

单例模式保证了一个类只有一个实例，并提供了一个全局访问点。
```java
public class Singleton {
  private static Singleton uniqueInstance;
  // other useful instance variables here

  private Singleton() {}

  public static synchronized Singleton getInstance() {  // synchronized 解决多线程可能返回不同对象的问题
    if (uniqueInstance == null) {
      uniqueInstance = new Singleton();
    }
    return uniqueInstance;
  }

  // Other useful methods here
}
```

但是同步是一种比较重的解决办法，因为当uniqueInstance被赋值以后，synchronized就不再被需要了，因此上面的写法引入了额外的开销。

一种办法是将 `Lazily created one` 变为 `Eagerly created one`。这种方法依赖JVM来在装载类的时候创建一个唯一的Singleton实例。JVM保证了实例的创建早于任何线程获取实例。
```java
public class Singleton {
  private static Singleton uniqueInstance = new Singleton();  // JVM创建优先于线程访问

  private Singleton() {}

  public static Singleton getInstance() {
    return uniqueInstance;
  }
}
```

另一种办法是使用“double-checked locking”来减少getInstance()中同步的使用。
```java
public class Singleton {
  private volatile static Singleton uniqueInstance; // volatile 保证了uniqueInstance在多线程之间的可见性

  private Singleton() {}

  public static Singleton getInstance() {
    if (uniqueInstance == null) {   // 保证了同步机制只在第一次实例化的时候被使用
      synchronized (Singleton.class) {
        if (uniqueInstance == null) {
          uniqueInstance = new Singleton();
        }
      }
    }
    return uniqueInstance;
  }
}
```

<h4>🔥 命令模式(The Command Pattern)</h4>

命令模式将请求封装为一个对象，从而允许您用不同的请求、队列或日志请求对其他对象进行参数化，并支持无法执行的操作。

![The  Command Pattern](/assets/201709/command_pattern.png)

<h4>🔥 适配器模式(The Adapter Pattern)</h4>

适配器模式将一个类的接口转换为客户端期望的另一个接口。适配器让类协同工作，否则由于接口不兼容而无法正常工作。

![The  Adapter Pattern](/assets/201709/adapter_pattern.png)

**对象适配器**（Object Adapter）使用组合（Composition，Adaptee作为Adapter的一部分）来实现。

![The Object Adapter Pattern](/assets/201709/object_adapter_pattern.png)

**类适配器**（Class Adapter）使用多继承（multiple inheritance）来实现，在允许多继承的语言中可以使用。

![The Class Adapter Pattern](/assets/201709/class_adapter_pattern.png)

<h4>🔥 外观模式(The Facade Pattern)</h4>

外观模式为子系统中的一组接口提供统一的接口，它定义了一个高层的接口使得子系统更易于使用。

![The Facade Pattern](/assets/201709/facade_pattern.png)

*适配器模式和外部模式的不同之处在于：适配器模式的目的是改变接口，使其与客户端期望的接口匹配。 Facade Pattern的目的是为子系统提供一个简化的接口。*

**适配器封装一个对象来改变它的接口，装饰器包装一个对象来添加新的行为和责任，而外观模式包装一组对象来进行简化。**

<h4>🔥 模板方法模式(The Template Method Pattern)</h4>

模板方法模式定义了方法中算法的框架，将一些步骤推送到子类。模板方法允许子类重新定义算法的某些步骤，而不改变算法的结构。

![The Template Method Pattern](/assets/201709/template_method_pattern.png)

```java
abstract class AbstractClass {
  final void templateMethod() {
    primitiveOperation1();
    primitiveOperation2();
    concreteOperation();
  }

  abstract void primitiveOperation1();
  abstract void primitiveOperation2();

  void concreteOperation() {
    // implementation here
  }
}
```

带钩子（hook）的模板方法模式:

```java
public abstract class CaffeineBeverageWithHook {
  final void prepareRecipe() {
    boilWater();
    brew();
    pourInCup();
    if (customerWantsCondiments()) {
      addCondiments();
    }
  }

  abstract void brew();
  abstract void addCondiments();

  void boilWater() {
    System.out.println("Boiling water");
  }

  void pourInCup() {
    System.out.println("Pouring into cup");
  }

  boolean customerWantsCondiments() { // 子类可以Override该方法，但不是必须的
    return true;
  }
}
```

*当子类必须提供算法中的方法或步骤的实现时，使用抽象方法。当算法的那部分是可选的时候使用钩子。使用钩子，一个子类可以选择实现该钩子，但它不是必须的。*


<br>
<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 Freeman, Eric,, Freeman, Elisabeth., Sierra, Kathy.Bates, Bert., eds. Head First Design Patterns. Sebastopol, CA : O'Reilly, 2004. Print.
</span>
