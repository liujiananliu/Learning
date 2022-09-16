# Java 面向对象

## 方法引用

> [方法引用](https://www.runoob.com/java/java8-method-references.html)

方法引用通过方法的名字来指向一个方法。

方法引用可以使语言的构造更紧凑简洁，减少冗余代码。

方法引用使用一对冒号 **::** 。

Java中的四种不同的方法引用：

1. 构造器引用
   ```java
   //语法是Class::new，或者更一般的Class< T >::new
   final Car car = Car.create( Car::new );
   final List< Car > cars = Arrays.asList( car );
   ```

2. 静态方法引用
   ```java
   //语法是Class::static_method
   cars.forEach( Car::collide );
   ```

3. 特定类的任意对象的方法引用
   ```java
   //语法是Class::method
   cars.forEach( Car::repair );
   ```

4. 特定对象的方法引用
   ```java
   //语法是instance::method
   final Car police = Car.create( Car::new );
   cars.forEach( police::follow );
   ```



## Lambda表达式

> [Lambda表达式](https://www.runoob.com/java/java8-lambda-expressions.html)

Lambda 表达式，也可称为闭包，它是推动 Java 8 发布的最重要新特性。

Lambda 允许把函数作为一个方法的参数（函数作为参数传递进方法中）。

使用 Lambda 表达式可以使代码变的更加简洁紧凑。



**lambda 表达式的语法格式如下：**

```java
(parameters) -> expression
或
(parameters) ->{ statements; }
```



**以下是lambda表达式的重要特征:**

- **可选类型声明：**不需要声明参数类型，编译器可以统一识别参数值。
- **可选的参数圆括号：**一个参数无需定义圆括号，但多个参数需要定义圆括号。
- **可选的大括号：**如果主体包含了一个语句，就不需要使用大括号。
- **可选的返回关键字：**如果主体只有一个表达式返回值则编译器会自动返回值，大括号需要指定表达式返回了一个数值。



**Lambda 表达式的简单例子:**

```java
// 1. 不需要参数,返回值为 5  
() -> 5  
  
// 2. 接收一个参数(数字类型),返回其2倍的值  
x -> 2 * x  
  
// 3. 接受2个参数(数字),并返回他们的差值  
(x, y) -> x – y  
  
// 4. 接收2个int型整数,返回他们的和  
(int x, int y) -> x + y  
  
// 5. 接受一个 string 对象,并在控制台打印,不返回任何值(看起来像是返回void)  
(String s) -> System.out.print(s)
```

