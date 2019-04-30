### 浅谈java与scala单例模式

##### 单例模式
> - 简单描述: 想让一个类只能构建一个对象的设计模式
> - 详细描述: 单例模式是一个单一的类负责创建自己的对象,并确保只有单个对象被创建.只提供一种访问其对象的方式,不需要实例化该类即可访问

##### java实现单例模式方式
````
//1.饿汉式
public class Singleton {  
    private static Singleton instance = new Singleton();  
    private Singleton (){}  
    public static Singleton getInstance() {  
    return instance;  
    }  
}

//2.登记式/静态内部类(支持lazy loading)
public class Singleton {  
    private static class SingletonHolder {  
    private static final Singleton INSTANCE = new Singleton();  
    }  
    private Singleton (){}  
    public static final Singleton getInstance() {  
    return SingletonHolder.INSTANCE;  
    }  
}
````
- 饿汉式实现方式基于classloader机制避免了多线程同步问题,但在类加载时就实例化instance,没有达到lazy loading的效果
- 登记式/静态内部类基于classloader机制避免了多线程同步问题,且只有在显式调用getInstance方法才显式加载SingletonHolder类,实现lazy loading的效果

##### scala实现单例模式方式
````
class Singleton(val v: String) {

  println("创建:" + this)

  override def toString(): String = "singleton类变量:" + v
}

object Singleton {

  def apply(v: String) = {

    new Singleton(v)
  }

  def main(args: Array[String]) = {
    println(Singleton("hello scala singleton"))
  }
  
}
````
- scala中是没有static的,但我们也可以通过关键字object实现单例模式.
- scala使用单例模式时,除了定义的类外,还需要定义一个同名的object对象,它和类的区别是object对象不能带参数
- 当单例对象与某个类共享同个名称时,他被称为这个类的**伴生对象(companion object)**,必须在同一个源文件定义类和它的伴生对象.类和它的伴生对象可以互相访问其私有对象


###### 相关资料链接:

- https://www.runoob.com/design-pattern/singleton-pattern.html
- https://www.runoob.com/scala/scala-classes-objects.html