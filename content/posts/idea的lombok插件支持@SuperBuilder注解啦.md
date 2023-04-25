---
title: "idea的lombok插件支持@SuperBuilder注解啦"
date: 2019-10-09 00:00:00.0
draft: false
type: "post"
showTableOfContents: true
tags: ["IDE"]
---

---
layout:     post
title:      "idea的lombok插件支持@SuperBuilder注解啦"
date:       2019-10-09
author:     "马赛琦"
tags:
    - idea
    - lombok

---

## 1. 前言

今早进公司打开idea，弹出更新提示，简单看了下，原来是idea的lombok插件更新了，惊喜的发现update log上写着`Add support for @SuperBuilder`。

![](https://img.masaiqi.com/2019-10-09-024620.png)

为什么说是惊喜呢？因为之前也有用到这个的场景，去官网认认真真看完了`@SuperBuilder`的用法以及描述，刚准备大展拳脚，结果发现idea上怎么写都识别不出来，后来去插件的github上看了一下，在issue中发现很多请求插件更新支持`@SuperBuilder`注解，而插件作者大概的回复就是已经在开发计划中了，不要催，催也不能提高进度。不得已，在自己的项目中只能冗余一些代码，而不能基于lombok更加优雅简洁的去写。

虽然插件已经更新支持，但是一直没有实际使用导致我都忘了很多用法了，这篇文章基于官网文档，用来记录与复习相关用法。

## 2. 关于@SuperBuilder

### 2.1. 首先了解`@Builder`

看到这篇文章的你肯定已经用过这个注解，这里简单陈述一下基本用法，如果你已经了解，可以略过此部分。

#### 2.1.1. 引入依赖(Maven结构)

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.8</version>
    <scope>provided</scope>
</dependency>
```

#### 2.1.2. 创建一个类

```java
/**
 * Ming
 *
 * @author sq.ma
 * @date 2019/10/9 上午9:37
 */
@Builder
public class Ming {
    private Integer age;
    private String name;
}
```

这里使用`@Builder`注解，就可以在创建新实例的时候这样写：

````java
Ming mingA = Ming.builder().build();
Ming mingB = Ming.builder()
  	.age(11)
  	.build();
Ming mingD = Ming.builder()
  	.age(11)
  	.name("小明")
  	.build();
````

可以看到，我们只要写一个`@Builder`注解，有如下好处：

- 一个注解代替若干参数情况下的构造函数，缩减了构造类的代码量
- 通过Builder构造的方式，即`.属性名(值)`这样的方式，比直接使用构造函数的方式更加具备可读性，比频繁使用set方法的方式更加简洁。

### 2.2. 了解`@SuperBuilder`

#### 2.2.1. `@SuperBuilder`解决了什么样的问题

在上文(2.1)中，我们了解了`@Builder`的使用，那么我们将例子中的`Ming`这个类的成员属性放到父类当中：

````java
/**
 * @author sq.ma
 * @date 2019/10/9 上午10:01
 */
public class Person {
    private Integer age;
    private String name;
}

@Builder
public class Ming extends Person{
}
````

这个时候，我们之前的调用的`.builder`都会报错，这是因为`@Builder`并不支持父类成员属性的构造，`@SuperBuilder`注解的出现，就是用来解决这个问题。

````java
/**
 * @author sq.ma
 * @date 2019/10/9 上午10:01
 */
@SuperBuilder
public class Person {
    private Integer age;
    private String name;
}

@SuperBuilder
public class Ming extends Person{
}
````

这样子类就可以正常获取到父类的成员属性进行builder构造了。

#### 2.2.2. `@SuperBuilder(toBuilder = true)`用法

`toBuilder`属性默认关闭，如果开启，则所有的父类应该也要开启，效果如下：

````java
Ming mingD = Ming.builder()
      .age(11)
      .name("小明")
      .build();
Ming mingF = mingD.toBuilder().name("猪").build();
System.err.println(mingD.toString());
System.err.println(mingF.toString());
````

![](https://img.masaiqi.com/2019-10-09-024651.png)

通过设置true，所有的类实例会拥有`toBuilder`方法，这是一个类似`深拷贝`的一个方法，不会改变原有实例的属性，生成一个新的实例。在`toBuilder`中有赋值的属性则会改变为赋值属性，没有赋值的以调用的实例中的值为准。

#### 2.2.3. `@SuperBuilder(buildMethodName = "execute", builderMethodName = "helloWorld", toBuilder = true)` 用法

这个用法其实没什么意思，就是自定义方法名，不展开赘述。

#### 2.2.4. `@Builder.ObtainVia(XXX)` 用法

这个是Filed或parameter的注解，我们看下源码

````java
@Target({FIELD, PARAMETER})
@Retention(SOURCE)
public @interface ObtainVia {
		/**
		 * @return Tells lombok to obtain a value with the expression {@code this.value}.
		 */
		String field() default "";
		
		/**
		 * @return Tells lombok to obtain a value with the expression {@code this.method()}.
		 */
		String method() default "";
		
		/**
		 * @return Tells lombok to obtain a value with the expression {@code 			   SelfType.method(this)}; requires {@code method} to be set.
		 */
		boolean isStatic() default false;
}
````

其中，

- `field`是告诉lombok赋值时从哪个属性取值
- `method`是告诉lombok赋值时调用什么方法
- `isStatic`是跟在method后的，默认为false，代表相应的`method`是否是静态

需要注意的是这几个方法只有在`ToBuilder = true`的时候有效，最好不要混合使用(有先后顺序问题)

#### 2.2.5. 注意补充构造方法

**2019-10-10 更新**

使用`@Builder`或`@SuperBuilder`注解时，不会默认创建空参构造函数，如果你有额外使用空参构造函数或全参构造函数的需求，需要在子类和父类都加上以下注解：

````
@AllArgsConstructor //全参构造函数
@NoArgsConstructor //空参构造函数
````

## 3. 参考文献

[@SuperBuilder - lombok官网](https://projectlombok.org/features/experimental/SuperBuilder)