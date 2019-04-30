### 不一样的apply方法

##### 才开始学scala的同学会对apply方法一脸茫然,其实只需要了解下scala的理念就清晰了
> - java是面向对象,所以我们需要先创建对象然后去能面向,嗯....
> - scala认为函数,变量都可以是对象(函数对象-所以scala是函数式编程语言),来着不拒嘛,嗯....
> - 问题来了,scala你抓到个啥都说是对象,请问咋调用对象的方法

- **将对象以函数对象调用时,scala会隐式地改为调用对象的apply方法,像调用XXX("hello")时调用的是XXX.apply("hello")**
###### 变量调用apply方法
````
val v="hello scala apply"
println(v.apply(1))
println(v(1)) //v(1) 等同v.apply(1)
````
###### 函数调用apply方法
````
val f=(x:Int,y:Int)=>x+y
println(f.apply(1,2))
println(f(1,2))  //f(1,2) 等同f.apply(1,2)
````
###### 对象调用apply方法
````
class Person(val name: String) {

  val _name = name

  def getName(): String = _name

  def apply(name: String) = println(name + " and " + _name)
}

object ApplyDemo {
  
  def main(args: Array[String]): Unit = {
    val o = new Person("zhangsan")
    o("lisi")
    o.apply("lisi")
  }
}
````


###### 相关资料链接:

- https://stackoverflow.com/questions/9737352/what-is-the-apply-function-in-scala/9738862#9738862