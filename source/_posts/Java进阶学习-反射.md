---
title: Java进阶学习-反射
categories: 学习笔记
date: 2019-02-04 23:46:20
tags: Java
---
# 简述
主要是自己对于Java一些进阶知识的补充学习，包括但不局限于反射，多线程、源码等，本篇主要介绍反射，相关API在Java.lang.reflect中。
<!--more-->

# 反射
反射是Java运行时动态获取类或者方法信息等一种机制。
任何一个类都是Class的实例对象，要获取这个实例对象，有以下三种方式；


```java
//实际上可以看出来每一个类里面都有一个隐含的成员变量class
Class c1 = Color.class;
//通过实例来获取类信息​
Color color = new Color();
Class c2 = color.getClass();
//第三种
Class c3 = null;
try{
    c3 = Class.forName("com.test.Color");
}
catch(ClassNotFoundExection e){
    e.printStackTrace();
}
//可以通过类的类类型创建该类的对象实例 ---> 通过c1 c2 c3创建新实例
try {
	AQSTest a = (AQSTest) c1.newInstance();//前提是必须有默认无参构造函数
} catch (InstantiationException | IllegalAccessException e) {
	// TODO Auto-generated catch block
	e.printStackTrace();
}
	
	//使用new 创建一个类的实例时，是静态加载类，即在编译时刻就需要加载所有的有可能使用到的类
try {
	//动态获取类，在运行时加载
	Class c4 = Class.forName(args[0]);
	//通过类类型，动态创建类
	c4.newInstance();
} catch (ClassNotFoundException e) {
	// TODO Auto-generated catch block
	e.printStackTrace();
} catch (InstantiationException e) {
	// TODO Auto-generated catch block
	e.printStackTrace();
	} catch (IllegalAccessException e) {
	// TODO Auto-generated catch block
	e.printStackTrace();
	}
	//基本的数据类型，以及void等关键字都有类类型
```

## 成员函数

```java
/**
 * 打印成员变量
 * @param obj
 */
public static void printFieldMessage(Object obj) {
	Class c = obj.getClass();
	Field[] fs = c.getDeclaredFields();
	/**
	 * 成员变量也是对象，是
	 * java.lang.refelct.Field的实例对象，该类封装了关于成员变量的操作
	 */
	for (Field field : fs) {
		//得到成员变量类型的类类型
		Class fieldType = field.getType();
		String typeName = fieldType.getName();
		//得到成员变量的名称
		String fieldName = field.getName();
		System.out.println(typeName+" "+fieldName);
	}
}
```

## 构造函数

```java
/**
 * 打印对象的构造函数的信息
 * @param obj
 */
public static void printConstructor(Object obj){
	Class c = obj.getClass();
	//获取所有的自己所有的public的构造函数
	Constructor[] cs = c.getDeclaredConstructors();
	for (Constructor constructor : cs) {
		System.out.print(constructor.getName()+"(");
		//获取构造函数的参数列表 ---> 得到的是参数列表的类类型
		Class[] parameterTypes = constructor.getParameterTypes();
		for (Class class1 : parameterTypes) {
			System.out.print(class1.getName()+",");
		}
		System.out.println(")");
	}
	System.out.println();
}
```
## 方法的反射操作

```java
public static void main(String[] args) {
	A a = new A();
	Class c = a.getClass();
	try {
		//获取方法对象
		Method m = c.getMethod("print", new Class[]{String.class,String.class});
		//c.getMethod("print", int.class,int.class);
        //方法的反射操作是用方法的对象m自己调用自己，和a.print()的效果相同
		//方法如果没有返回值返回null，有返回值返回具体的返回值
		Object obj = m.invoke(a, "1","2");
	} catch (Exception e) {
		// TODO Auto-generated catch block
		e.printStackTrace();
	} 
}

Class A{
	
	public void print(int a,int b){
		System.out.println(a+b);
	}
	
	public void print(String a,String b){
		System.out.println(a.toUpperCase()+" "+b.toUpperCase());
	}
}
```
## 通过反﻿﻿射认识范型

```java

public static void main(String[] args) {
	//首先看一个问题：
	List<String> list = new ArrayList<String>();
	List list1 = new ArrayList<>();
	//list.add(20);错误操作
	Class c = list.getClass();
	Class c1 = list1.getClass();
	System.out.println(c == c1);//true
	//反射的操作都是编译之后的操作
	
	/**
	 * 结论：
	 * 说明编译之后集合的泛型是去泛型化的，Java中集合的泛型，是为了防止错误输入的
	 * 只在编译阶段有效
	 */
	try {
		Method m = c.getMethod("add", Object.class);
		m.invoke(list, 20);
		System.out.println(list.size());
		System.out.println(list);
		//用string进行遍历时就会进行报错
		/*for (String string : list) {
			System.out.println(string);
		}*/
		for (Object object : list) {
			System.out.println(object);
		}
	} catch (Exception e) {
		e.printStackTrace();
	}
}
```
