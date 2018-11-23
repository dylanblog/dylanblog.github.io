---
layout: post
title: "Bridge策略模式"
subtitle: ''
author: "DylanYu"
header-style: text
tags:
  - 设计模式
  - 桥接模式
---

桥接（Bridge）是用于把**抽象化与实现化解耦**，使得二者可以独立变化。这种类型的设计模式属于结构型模式，它通过提供抽象化和实现化之间的桥接结构，来实现二者的解耦。

这种模式涉及到一个作为桥接的接口，使得实体类的功能独立于接口实现类。这两种类型的类可被结构化改变而互不影响。

面向对象的程序设计（OOP）里有类继承（子类继承父类）的概念，如果一个类或接口有多个具体实现子类，如果这些子类具有以下特性：
- 存在相对并列的子类属性。
- 存在概念上的交叉。
- 可变性。
我们就可以用Bridge模式来对其进行抽象与具体，对相关类进行重构。

> 桥接是一个接口，它与一方应该是绑定的，也就是解耦的双方中的一方必然是继承这个接口的，这一方就是实现方，
> 而另一方正是要与这一方解耦的抽象方，如果不采用桥接模式，一般我们的处理方式是直接使用继承来实现，这样双方之间处于强链接，
> 类之间关联性极强，如要进行扩展，必然导致类结构急剧膨胀。采用桥接模式，正是为了避免这一情况的发生，将一方与桥绑定，
> 即实现桥接口，另一方在抽象类中调用桥接口（指向的实现类），这样桥方可以通过实现桥接口进行单方面扩展，而另一方可以继承抽象类而单方面扩展，
> 而之间的调用就从桥接口来作为突破口，不会受到双方扩展的任何影响。

例子：
    笔有铅笔、钢笔、毛笔、水笔等等；图形有：直线、圆、三角形、树叶等等。各自独立变化。

```
/** "Implementor" */
interface DrawingAPI {
    public void drawCircle(double x, double y, double radius);
}

/** "ConcreteImplementor"  1/2 */
class DrawingAPI1 implements DrawingAPI {
    public void drawCircle(double x, double y, double radius) {
        System.out.printf("API1.circle at %f:%f radius %f\n", x, y, radius);
    }
}

/** "ConcreteImplementor" 2/2 */
class DrawingAPI2 implements DrawingAPI {
    public void drawCircle(double x, double y, double radius) {
        System.out.printf("API2.circle at %f:%f radius %f\n", x, y, radius);
    }
}

/** "Abstraction" */
abstract class Shape {
    protected DrawingAPI drawingAPI;

    protected Shape(DrawingAPI drawingAPI){
        this.drawingAPI = drawingAPI;
    }

    public abstract void draw();                             // low-level
    public abstract void resizeByPercentage(double pct);     // high-level
}

/** "Refined Abstraction" */
class CircleShape extends Shape {
    private double x, y, radius;
    public CircleShape(double x, double y, double radius, DrawingAPI drawingAPI) {
        super(drawingAPI);
        this.x = x;  this.y = y;  this.radius = radius;
    }

    // low-level i.e. Implementation specific
    public void draw() {
        drawingAPI.drawCircle(x, y, radius);
    }
    // high-level i.e. Abstraction specific
    public void resizeByPercentage(double pct) {
        radius *= pct;
    }
}

/** "Client" */
class BridgePattern {
    public static void main(String[] args) {
        Shape[] shapes = new Shape[] {
            new CircleShape(1, 2, 3, new DrawingAPI1()),
            new CircleShape(5, 7, 11, new DrawingAPI2()),
        };

        for (Shape shape : shapes) {
            shape.resizeByPercentage(2.5);
            shape.draw();
        }
    }
}
```