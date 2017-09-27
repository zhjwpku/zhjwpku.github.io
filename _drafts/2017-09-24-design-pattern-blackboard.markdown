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


<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 Freeman, Eric,, Freeman, Elisabeth., Sierra, Kathy.Bates, Bert., eds. Head First Design Patterns. Sebastopol, CA : O'Reilly, 2004. Print.
</span>
