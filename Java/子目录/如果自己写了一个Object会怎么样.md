# 假如我自己写了一个Object类会怎么样？
最近去面试遇到了这样一个问题，如果我自己写了一个Object类，在程序里用的话会发生什么？

这个问题引起了我的思考，xjb回答一通后想到了双亲委派模型等各种看到过的东西，不过实践还是硬道理，所以在这里记录一下。
随便写一个Object类放在一个随意的包名下：

````java
public class Object {
    int a = 0;
    @Override
    public String toString() {
        return "Object [a=" + a + "]";
    }
    public static void main(String[] args) {
        Object o1 = new Object();
        System.out.println(o1);
        java.lang.Object o2 = new java.lang.Object();
        System.out.println(o2);
    }
}
````



输出的结果是

`````
Object [a=0]
java.lang.Object@15db9742
`````



很显然在IDE里自己写一个Object类是可以正常使用的，只不过由于这个类与JAVA自己的Object类重名，所以想要用JAVA的Object类就需要使用全限定类名。

那我如果把自己写的Object放到java.lang包下会怎么样？

结果就是在我想生成一个toString()的时候eclipse会给我报个错，然后生成失败。
这怎么能难倒我呢，直接把上面写的toString()复制过来呀！
结果是@override注解会报错，看来IDE已经把这个类认为是JAVA自己的类了吧，毕竟我自己就是所有类的超类还怎么覆盖呢？

那我就写个主函数运行一下试试吧

````java
public class ObjectTest {
     public static void main(String[] args) {
        System.out.println("cc");
    }
}
````

然后淬不及防的JVM报错
	![JVM1](image\SouthEast)

​	![JVM2](image\SouthEast1)

![控制台报错](image\SouthEast2)

可以看出来是ClassLoader 在加载类的过程中抛出了SecurityException异常。
如何理解这时候发生了什么？？搬出来《深入理解JAVA虚拟机（第2版）》：

`````java
类从被加载到虚拟机内存中开始，到卸载出内存为止，它的整个生命周期包括：加载（Loading）、验证（Verrfication）、准备（Preparation）、解析（Resolution）、初始化（Initialization）、使用（Using）和卸载（Unloading） 7个阶段。其中验证、准备、解析3个部分统称为连接（Linking）。

在加载阶段，虚拟机需要完成以下3件事情：

1. 通过一个类的全限定名来获取定义此类的二进制字节流。
2. 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。
3. 在内存中生成一个代表这个类的java.lang.Class对象，作为方法去这个类的各种数据结构的访问入口。
`````

就第一条来说，可以从zip包里读取，这是jar，war，ear等格式的基础；也可以从网络中获取，如applet；或者运行时计算生成，就是动态代理技术。下面也着重来讨论一下第一条。

类加载器：其实就是第一件事情（通过一个类的全限定名来获取定义此类的二进制字节流），实现这个动作的代码模块就称为类加载器。
可以用在：类层次划分、OSGi、热部署、代码加密等领域。
对于任意一个类，都需要由加载它的类加载器和这个类本身一同确立其在Java虚拟机中的唯一性。
比如自定义一个类加载器加载一个类，与系统加载器加载的类，即使是来自同一个Class文件，因为类加载器不同，依然是两个独立的类，instanceof结果是false；

````java
从JAVA虚拟机的角度来看，只存在两种不同的类加载器：一种是启动类加载器（Bootstrap ClassLoader），属于虚拟机自身的一部分（HotSpot是用c++实现的）；另一种就是所有其他的类加载器，这些类加载器都由Java语言实现，独立于虚拟机外部，并且全部都继承自抽象类java.lang.ClassLoader。

大部分的程序都会使用到以下三种类加载器：
启动类加载器（Bootstrap ClassLoader）：负责加载< JAVA_HOME >\lib目录中的或者-Xbootclasspath参数指定的路径中的虚拟机识别的类库 到内存中。（识别按文件名识别，比如rt.jar）.
扩展类加载器（Extension ClassLoader）：负责< JAVA_HOME >\lib\ext目录中的或者被java.ext.dirs系统变量所指定的路径中的所有类库，开发者可以直接使用。
应用程序类加载器（Application ClassLoader）：也称为系统类加载器，可以由ClassLoader中的getSystemClassLoader()方法获得，负责加载ClassPath上所指定的类库，开发者可以直接使用，一般情况下是程序中默认的类加载器。
````

<img src="image\image-20220714180948324.png" alt="image-20220714180948324" style="zoom: 50%;" />


双亲委派模型：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传送到顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需的类）时，子加载器才会尝试自己去加载。

所以回到我们原来的问题，当我们自己写了一个java.lang.Object类并放在ClassPath中，应用程序类加载器委托扩展类加载器，扩展类加载器再委托启动类加载器，因为后两个路径中都没有这个class，所以应用类加载器会去尝试加载这个类。
但是，真正的java.lang.Object类已经被启动类加载器加载到了虚拟机内存中，如果应用类加载器也成功加载的话，那将会出现多个不同的Object类，程序就会变的一片混乱。。

结论：再来看这个报错：

![控制台报错](https://img-blog.csdn.net/20170921112306505?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd2FuZ2NjcGFs/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

倒数第三行就已经表明了应用程序类加载器在尝试加载这个类，但是因为这个禁止使用的包名，java.lang，所以加载失败抛出了SecurityException。
而不在java.lang包下的类就可以随意的通过这个应用程序类加载器加载使用了。

这时候还有接下来一个问题，

“如果我自己写的Object类放在rt.jar里并且就放在< JAVA_HOME >\bin”目录下会怎么样？

实际操作一下，我们替换rt.jar里的Object.class文件为我们自己写的Object，

![这里写图片描述](image\SouthEast45)

可以看出，jvm确实是只根据文件名来识别，成功加载了我们的Object类。。只不过由于缺少一些方法而报错了。
AMAZING！
