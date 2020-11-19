## Mybatis动态sql是做什么的？都有哪些动态sql？简述一下动态sql的执行原理？

Mybatis动态sql可以根据输入的参数动态拼接SQL语句。

动态sql包含以下类型
- if：用于简单的条件判断
- where：用于解决SQL语句中where关键字以及条件中第一个and或者or的问题
- set：动态前置 SET 关键字，同时也会消除无关的逗号，因为用了条件语句之后很可能就会在生成的赋值语句的后面留下这些逗号。
- trim：在条件判断完的SQL语句前后 添加或者去掉指定的字符
- choose、when、otherwise：用于分支判断，类似于java中的switch case,只会满足所有分支中的 一个
- foreach：遍历集合或数组
- bind：从 OGNL 表达式中创建一个变量并将其绑定到上下文

执行原理：

1. 在解析select|insert|update|delete标签时，XMLConfigBuilder会调用XMLStatementBuilder.parseStatementNode解析每个select|insert|update|delete标签

   ```java
     private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
       for (XNode context : list) {
         final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
         try {
           statementParser.parseStatementNode();
         } catch (IncompleteElementException e) {
           configuration.addIncompleteStatement(statementParser);
         }
       }
     }
   ```

   

2. XMLStatementBuilder调用XMLLanguageDriver.createSqlSource来创建SqlSource

   ```java
   SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
   ```

3. XMLLanguageDriver.createSqlSource创建XMLScriptBuilder并调用其parseScriptNode方法

   ```java
     public SqlSource createSqlSource(Configuration configuration, XNode script, Class<?> parameterType) {
       XMLScriptBuilder builder = new XMLScriptBuilder(configuration, script, parameterType);
       return builder.parseScriptNode();
     }
   ```

   

4. XMLScriptBuilder.parseScriptNode会调用parseDynamicTags方法解析动态标签。如果发现动态标签则将标记为动态SQL语句，并使用标签对应的Handler添加到MixedSqlNode中

   ```java
     public SqlSource parseScriptNode() {
       MixedSqlNode rootSqlNode = parseDynamicTags(context);
       SqlSource sqlSource = null;
       if (isDynamic) {
         sqlSource = new DynamicSqlSource(configuration, rootSqlNode);
       } else {
         sqlSource = new RawSqlSource(configuration, rootSqlNode, parameterType);
       }
       return sqlSource;
     }
   ```

5. 在执行SQL时，会调用MixedSqlNode的apply方法遍历所有添加到MixedSqlNode的SQLNode并调用其apply方法处理动态标签

   ```java
     @Override
     public boolean apply(DynamicContext context) {
       for (SqlNode sqlNode : contents) {
         sqlNode.apply(context);
       }
       return true;
     }
   ```

   

## Mybatis是否支持延迟加载？如果支持，它的实现原理是什么？

支持。实现原理：使用动态代理创建代理对象。当调用代理对象的延迟加载属性的get方法时，代理对象会拦截该方法，用事先保存好的SQL查询数据库加载数据。

## Mybatis都有哪些Executor执行器？它们之间的区别是什么？

- SimpleExecutor：简单的 Executor。每次开始读或写操作，都创建对应的 Statement 对象。执行完成后，关闭该 Statement 对象。
- ReuseExecutor：可重用的 Executor。每次开始读或写操作，优先从缓存中获取对应的 Statement 对象。如果不存在，才进行创建。执行完成后，不关闭该 Statement 对象。
- BatchExecutor：批量执行的 Executor 
- CachingExecutor：支持二级缓存的 Executor 的实现类

## 简述下Mybatis的一级、二级缓存（分别从存储结构、范围、失效场景。三个方面来作答）？

一级缓存：使用了HashMap作为存储结构。它的作用域是SqlSession。当查询到的数据，进行增删改的操作的时候，缓存将会失效。
二级缓存：默认实现使用的也是HashMap，如果使用redis作为二级缓存，则使用的是redis的hash结构。二级缓存是基于mapper文件的namespace的。当开启flushCache时，在mapper的同一个namespace中，如果有其它insert、update, delete操作数据后缓存会失效。如果不开启flushCache会出现脏读。

## 简述Mybatis的插件运行原理，以及如何编写一个插件？

1. 运行原理：mybatis可以编写针对Executor、StatementHandler、ParameterHandler、ResultSetHandler四个接口的插件，mybatis使用JDK的动态代理为需要拦截的接口生成代理对象，然后实现接口的拦截方法，所以当执行需要拦截的接口方法时，会进入拦截方法（AOP面向切面编程的思想）
2. 编写插件

- 编写Intercepror接口的实现类
- 设置插件的签名，告诉mybatis拦截哪个对象的哪个方法
- 最后将插件注册到全局配置文件中

```java
//插件签名，告诉mybatis当前插件拦截哪个对象的哪个方法
//type表示要拦截的目标对象，method表示要拦截的方法，args表示要拦截方法的参数
@Intercepts({
	@Signature(type=StatementHandler.class,method="parameterize",args=java.sql.Statement.class)
})
public class MyInterceptor implements Interceptor {

	//拦截目标对象的目标方法执行
	@Override
	public Object intercept(Invocation invocation) throws Throwable {
		System.out.println("MyInterceptor 拦截目标对象："+invocation.getTarget()+"的目标方法："+invocation.getMethod());
		
		/*
		 * 插件的主要功能：在执行目标方法之前，可以对sql进行修改已完成特定的功能
		 * 例如增加分页功能，实际就是给sql语句添加limit；还有其他等等操作都可以
		 * */
		
		//执行目标方法
		Object proceed = invocation.proceed();
		//返回执行后的返回值
		return proceed;
	}
	//包装目标对象：为目标对象创建代理对象
	@Override
	public Object plugin(Object target) {
		System.out.println("MyInterceptor 为目标对象"+target+"创建代理对象");
		//this表示当前拦截器，target表示目标对象，wrap方法利用mybatis封装的方法为目标对象创建代理对象（没有拦截的对象会直接返回，不会创建代理对象）
		Object wrap = Plugin.wrap(target, this);
		return wrap;
	}
	//设置插件在配置文件中配置的参数值
	@Override
	public void setProperties(Properties properties) {
		System.out.println("MyInterceptor 配置的参数："+properties);
	}

}
```

```xml
<plugins
	<plugin interceptor="com.mybatis_demo.plugin.MyInterceptor"></plugin>
</plugins>
```

