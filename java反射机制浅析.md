**基本概念**

​	Java反射机制是在运行状态时，对于任意一个类，都能够获取到这个类的所有属性和方法，对于任意一个对象，都能够调用它的任意一个方法和属性(包括私有的方法和属性)，这种动态获取的信息以及动态调用对象的方法的功能就称为java语言的反射机制。

首先获取class实例，三种方法。

```
1.调用Class类的静态方法forName获取某个类的Class类实例：
Class aa = Class.forName("com.bb.User");
2.访问某个类的class属性，这个属性就存储着这个类对应的Class类的实例：
Class aa = com.bb.User.class;
3.调用某个对象的getClass()方法：
obj.getClass() 如果上下⽂中存在某个类的实例 obj ，那么我们可以直接通过obj.getClass() 来获取它的类
Class aa = (new User()).getClass();
```

通过Class对象获取某个方法：

```
aa.getMethod(方法名，这个方法的参数类型)
例：
Method method = aa.getMethod("setName", String.class);
```

然后去调用方法：

```
method.invoke((new User()), "bb");
或者
User user = new User();
\#class方法在第一步的时候就已经获取到
method.invoke(user, "bb");
```

**漏洞利用**

​	为什么要搞反射？Java安全机制会对代码的执行进行限制，例如限制代码的访问权限、限制代码的资源使用等。如果代码需要执行一些危险的操作，例如执行系统命令，就需要获取Java的安全权限。获取Java的安全权限需要经过一系列的安全检查，例如检查代码的来源、检查代码的签名等。如果代码没有通过这些安全检查，就无法获取Java的安全权限，从而无法执行危险的操作。然而，反射机制可以绕过Java安全机制的限制，比如可以访问和修改类的私有属性和方法，可以调用类的私有构造方法，可以创建和访问动态代理对象等。这些操作都是Java安全机制所禁止的，但是反射机制可以绕过这些限制，从而执行危险的操作。

说白了，反射就是为了绕过某些沙盒，有些沙盒禁止执行命令，通过反射可以绕过该限制。

**forName**

forName有两个函数重载：

```
 Class<?> forName(String name)
 Class<?> forName(String name, **boolean** initialize, ClassLoader loader)
```

第⼀个就是我们最常⻅的获取class的⽅式，其实可以理解为第⼆种⽅式的⼀个封装：

```
Class.forName(className)
// 等于
Class.forName(className, true, currentLoader)
```

​	默认情况下， forName 的第⼀个参数是类名；第⼆个参数表示是否初始化；第三个参数就是 ClassLoader 。ClassLoader 是⼀个“加载器”，告诉Java虚拟机如何加载这个类。Java默认的 ClassLoader 就是根据类名来加载类，这个类名是类完整路径，如 java.lang.Runtime 因此forName自动初始化。在正常情况下，除了系统类，如果我们想拿到一个类，需要先 import 才能使用。而使用forName就不需要，这样对于我们的攻击者来说就十分有利，我们可以加载任意类。

```
package org.example;
import java.io.BufferedReader;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.lang.reflect.Method;
public class Main {
  public static void main(String[] args) throws Exception {
​      Class<?> runtimeClass = Class.forName("java.lang.Runtime");
​      Method execMethod = runtimeClass.getMethod("exec", String.class);
​      Process process = (Process) execMethod.invoke(Runtime.getRuntime(), "calc.exe");
​      InputStream in = process.getInputStream();
​      BufferedReader reader = new BufferedReader(new InputStreamReader(in));
​      String line;    
​      while ((line = reader.readLine()) != null) {
​      System.out.println(line);    
​      }  
  }
}
```

**getMethod** 

​	getMethod 的作用是通过反射获取一个类的某个特定的公有方法。一般使用getMethod("exec", String.class) 来获取 Runtime.exec 方法。

还有一个获取方法的方法：getDeclaredMethod

两者区别：

getDeclaredMethod：获取当前类的所有声明的方法，包括public、[protected](https://so.csdn.net/so/search?q=protected&spm=1001.2101.3001.7020)和private修饰的方法。需要注意的是，这些方法一定是在当前类中声明的，从父类中继承的不算，实现接口的方法由于有声明所以包括在内。

getMethod：获取当前类和父类的所有public的方法。这里的父类，指的是继承层次中的所有父类。比如说，A继承B，B继承C，那么B和C都属于A的父类。

获取有参的方法时后面需要加参数类型，例：

```
Method declaredMethod = this.getClass().getDeclaredMethod("updateTopic", HttpServletRequest.class, HttpServletResponse.class);
```

**invoke**

invoke 的作用是执行方法，它的第一个参数是：

如果这个方法是一个普通方法，那么第一个参数是类对象

如果这个方法是一个静态方法，那么第一个参数是类

这也比较好理解了，我们正常执行方法是 [1].method([2], [3], [4]...) ，其实在反射里就是method.invoke([1], [2], [3], [4]...) 

invoke的第一个参数需要是实例

**私有方法的反射**

私有方法的反射通过两种方法一种是通过静态方法去调用另一种是通过getDeclaredConstructor 来获取这个私有的构造方法来实例化对象，进而执行命令

当我们需要调用私有方法时是不能直接调用的，invoke不能直接执行，例如下面这种情况：

```
Class clazz = Class.forName("java.lang.Runtime");
clazz.getMethod("exec", String.class).invoke(clazz.newInstance(), "id");
```

第一种：

直接调用Runtime类，会导致代码报错，因为Runtime类的构造方法是私有的，私有类方法涉及到很常见的设计模式：“单例模式。

因为有的方法只使用一次，一般都会将使用的类的构造函数设置为私有，然后编写一个静态方法来获取

Runtime类就是单例模式，我们只能通过 Runtime.getRuntime() 来获取到 Runtime 对象。我们将上述Payload进行修改即可正常执行命令了：

```
Class clazz = Class.forName("java.lang.Runtime");
clazz.getMethod("exec",String.class).invoke(clazz.getMethod("getRuntime").invoke(clazz),"calc.exe");
```

解释一下：

```
Class clazz = Class.forName("java.lang.Runtime");
Method execMethod = clazz.getMethod("exec", String.class);
Method getRuntimeMethod = clazz.getMethod("getRuntime");
Object runtime = getRuntimeMethod.invoke(clazz);
execMethod.invoke(runtime, "calc.exe");
```

第二种：

直接用 getDeclaredConstructor 来获取这个私有的构造方法来实例化对象，进而执行命令：

```
Class clazz = Class.forName("java.lang.Runtime");
Constructor m = clazz.getDeclaredConstructor();
m.setAccessible(true);
clazz.getMethod("exec", String.class).invoke(m.newInstance(), "calc.exe");
```

 