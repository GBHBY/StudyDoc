# Spring 

## AOP

- AOP：面向切面编程，过滤器也是一种AOP，AOP是对传统面向对象编程（OOP）的补充
- AOP主要作用是：通过配置可以实现业务逻辑和系统服务分离。比如：事务控制、日志、权限等
- AOP实现原理：**动态代理**，在运行期间对方法进行增强，不会产生新类
- 底层原理
  - **jdk动态代理**，被代理类需要实现接口（AOP默认使用jdk动态代理）
    - jdk反射机制生成一个代理接口的匿名类，调用具体方法的时候使用invokerHandler
    - 核心：`InvocationHandler`接口和`Proxy`类
  - **cglib**动态代理，不需要继承任何接口，cglib包在Spring core内，但如果某个类被标记为final，就无法使用cglib做动态代理

##  IOC：Inversion Of Contro

- IOC：控制反转
- 传统创建对象是通过new来创建，而IOC是利用**反射**的原理将对象创建的权力交给Spring
- Spring在运行的额时候根据配置文件来**动态**的创建对象和维护对象之间的关系
- 实现方式： 
  - 配置文件
  - 注解

### 实现原理

1. 创建xml配置文件
2. 通过dom4j解析xml
3. 工厂模式
4. 反射

- 首先创建xml文件
- 再通过dom4j解析xml文件，通过id值得到类的全限定类名
- 用反射来创建对象



### 创建Bean对象的三种方式：

- 使用默认函数构造器，类中不可以存在含参构造函数，否则报错

  - ```xml
     <bean id="accountService" class="com.GYB.service.impl.AccountServiceImpl" />
    ```

- 使用普通工厂来创建对象

  - ```xml
     <bean id="instanceFactory" class="com.GYB.factory.InstanceFactory" />
     <bean id="accountService"  factory-bean="instanceFactory" factory-method="getAccountService" />
    ```

  - ```java
    package com.GYB.factory;
    
    import com.GYB.service.IAccountService;
    import com.GYB.service.impl.AccountServiceImpl;
    
    public class InstanceFactory {
        public IAccountService getAccountService() {
            return new AccountServiceImpl();
        }
    }
    
    ```

- 使用静态工厂创建

  - ```xml
     <bean id="accountService" class="com.GYB.factory.StaticFactory" factory-method="getAccountService" />
    ```

  - ```java
    package com.GYB.factory;
    
    import com.GYB.service.IAccountService;
    import com.GYB.service.impl.AccountServiceImpl;
    
    public class StaticFactory {
        public static IAccountService getAccountService() {
            return new AccountServiceImpl();
        }
        
    }
    
    ```

### Spring默认创建的对象是单例的

- ```xml
  <!--bean的作用范围-->
          <!--用bean的scope标签来调整，属性如下：-->
              <!--singleton: 单例的，默认值-->
              <!--prototype：多例的-->
              <!--request：作用于web应用的请求范围-->
              <!--session：作用域web的会话范围-->
              <!--global-session：作用于集群环境的会话范围，当不是集群环境时，这个值就是session-->
  
      <!--bean的生命周期-->
          <!--单例对象-->
              <!--一一旦解析完配置文件，对象就会产生对象，只要容器存在，对象就存在-->
  <bean id="accountService" class="com.GYB.factory.StaticFactory" factory-method="getAccountService" scope="singleton" init-method="init" destroy-method="destroy" />
  <!--上面这段代买中的init-method和destroy-method标签很容易理解，不再赘述，但是destroy-method中的方法要想执行，必须有下面这段代码-->
  ```

### 注入Spring的两种方式

- set方法
- 构造函数注入
  - 具体去看SSM-Spring day01-4
- 复杂类型注入
  - 具体去看SSM-Spring day01-4

##  **SpringMVC执行流程:**

1. 用户发送请求至前端控制器DispatcherServlet
2. DispatcherServlet收到请求调用处理器映射器HandlerMapping。
3. 处理器映射器根据请求url找到具体的处理器，生成处理器执行链HandlerExecutionChain(包括处理器对象和处理器拦截器)一并返回给DispatcherServlet。
4. DispatcherServlet根据处理器Handler获取处理器适配器HandlerAdapter执行HandlerAdapter处理一系列的操作，如：参数封装，数据格式转换，数据验证等操作
5. 执行处理器Handler(Controller，也叫页面控制器)。
6. Handler执行完成返回ModelAndView
7. HandlerAdapter将Handler执行结果ModelAndView返回到DispatcherServlet
8. DispatcherServlet将ModelAndView传给ViewReslover视图解析器
9. ViewReslover解析后返回具体View
10. DispatcherServlet对View进行渲染视图（即将模型数据model填充至视图中）。
11. DispatcherServlet响应用户。



## 事务特性

原子性（Atomicity）

- 原子性是指事务是一个不可分割的工作单位，事务中的操作要么都发生，要么都不发生。
  比如：转账转过去的加和转时候的减必须一次发生

一致性（Consistency）

- 事务必须使数据库从一个一致性状态变换到另外一个一致性状态。
  比如：转账时双方的总数在转账的时候保持一致

隔离性（Isolation）

- 事务的隔离性是多个用户并发访问数据库时，数据库为每一个用户开启的事务，不能被其他事务的操作数据所干扰，多个并发事务之间要相互隔离。
  比如：多个用户操纵，防止数据干扰，就要为每个客户开启一个自己的事务；

持久性（Durability）

- 持久性是指一个事务一旦被提交，它对数据库中数据的改变就是永久性的，接下来即使数据库发生故障也不应该对其有任何影响。
  比如：如果我commit提交后 无论发生什么都 都不会影响到我提交的数据；

## Spring的四个隔离级别

1. **ISOLATION_DEFAULT** 这是一个PlatfromTransactionManager默认的隔离级别，使用数据库默认的事务隔离级别.另外四个与JDBC的隔离级别相对应 
2. **ISOLATION_READ_UNCOMMITTED** 这是事务最低的隔离级别，它充许别外一个事务可以看到这个事务未提交的数据。这种隔离级别会产生脏读，不可重复读和幻像读

3. **ISOLATION_READ_COMMITTED** 保证一个事务修改的数据提交后才能被另外一个事务读取。另外一个事务不能读取该事务未提交的数据。这种事务隔离级别可以避免脏读出现，但是可能会出现不可重复读和幻像读。

4. ISOLATION_REPEATABLE_READ 这种事务隔离级别可以防止脏读，不可重复读。但是可能出现幻像读。它除了保证一个事务不能读取另一个事务未提交的数据外，还保证了避免下面的情况产生(不可重复读)。

5. **ISOLATION_SERIALIZABLE** 这是花费最高代价但是最可靠的事务隔离级别。事务被处理为顺序执行。除了防止脏读，不可重复读外，还避免了幻像读。

 ## Spring的七个传播机制

1. **PROPAGATION_REQUIRED** 如果存在一个事务，则支持当前事务。如果没有事务则开启一个新的事务。

2. **PROPAGATION_SUPPORTS** 如果存在一个事务，支持当前事务。如果没有事务，则非事务的执行。但是对于事务同步的事务管理器，PROPAGATION_SUPPORTS与不使用事务有少许不同。

3. **PROPAGATION_MANDATORY** 如果已经存在一个事务，支持当前事务。如果没有一个活动的事务，则抛出异常。

4. **PROPAGATION_REQUIRES_NEW** 总是开启一个新的事务。如果一个事务已经存在，则将这个存在的事务挂起。

5. **PROPAGATION_NOT_SUPPORTED** 总是非事务地执行，并挂起任何存在的事务。

6. **PROPAGATION_NEVER** 总是非事务地执行，如果存在一个活动事务，则抛出异常

7. **PROPAGATION_NESTED**如果一个活动的事务存在，则运行在一个嵌套的事务中. 如果没有活动事务, 则按TransactionDefinition.PROPAGATION_REQUIRED 属性执行





