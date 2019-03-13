---
title: Android学习笔记一
categories: 学习笔记
date: 2019-03-09 09:06:59
tags:
- Android
keywords: [Android]
description: 学习《第一行代码》所记笔记
---
# 基本架构 #
## Linux内核层 ##
提供各种硬件驱动，显示、音频、相机、蓝牙驱动等。

## 系统运行时库 ##
系统特性支持，如SQLite数据库，OpenGL|ES库提供了3D绘图支持，WebKit提供浏览器内核支持。

还有Android运行时环境，主要提供了一些核心库，该环境中还有Dalvik虚拟机或者ART虚拟机。

## 应用层框架 ##
给运行于系统之上的APP提供API，系统内置软件和开发者开发的软件都是条用应用层的框架。

## 应用层 ##
用户下载的软件就是所处该层面。

<!--more-->

# 四大组件 #
## Activity：活动 ##
直接与用户进行交互的界面，也就是能看到的都是处于Activity中。

## Service：服务 ##
可以在用户退出应用后依然保持运行，例如音乐播放。

## Brodcast Provider：广播接收器 ##
接受电话短信，或者厂商的消息推送等。

## Content Provider：内容提供器 ##
应用程序间共享，比如你下载某个软件需要读取系统电话簿中的联系人。

# Activity #

## 新建活动 ##
创建Activity的步骤如下：

1. 首先创建视图，即布局文件layout；
2. 给activity中加载layout；
3. 在manifest文件中注册活动。

如下：
创建布局：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical" android:layout_width="match_parent"
    android:layout_height="match_parent">

    <!--定义活动1：创建活动布局-->
    <Button
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:id="@+id/button_1"
        android:text="点我点我"/>

</LinearLayout>
```
加载布局：

```java
@Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        //创建活动2：给活动加载布局，通常情况下这里传入一个布局文件的ID，资源目录下添加的资源都会在R文件中生成一个资源ID
        setContentView(R.layout.first_layout);
        //将取出来的view类型向下转型为Button类型
        final Button button = (Button) findViewById(R.id.button_1);
        button.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                //静态方法创造一个Toast对象，并调用show方法进行显示，第一个参数是上下文（本例子中活动本身就是一个context对象），
                //第二个参数是显示的文本，第三个是显示时长，有两个内置常量可以选择
                Toast.makeText(FirstActivity.this, "谢谢点击",Toast.LENGTH_SHORT).show();
                //执行下面的方法会销毁活动
                //finish();
            }
        });
    }
```
注册活动：

```xml
<!--创建活动3：在manifest文件中注册活动，给主活动指定的label不仅会成为标题中的内容，也会成为启动器中显示的名称-->
    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <!--外围指定包名了所以这里可以使用这种方式-->
        <activity android:name=".FirstActivity" >
            <intent-filter>
                <!--标记是一个主要的活动-->
                <action android:name="android.intent.action.MAIN"></action>
                <!--表明首先启动这个活动-->
                <category android:name="android.intent.category.LAUNCHER"></category>
            </intent-filter>
        </activity>
    </application>
```
## 新建menu ##
步骤：

1. 新建菜单布局文件；
2. 创建菜单；
3. 对菜单事见进行响应。

新建菜单布局文件：

```xml
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android">

    <!--定义菜单1：每个item都是一个菜单项-->
    <item
        android:id="@+id/add_item"
        android:title="Add"
        />
    <item
        android:id="@+id/remove_item"
        android:title="Remove"
        />

</menu>
```

创建菜单-在活动中重写onCreateOptionsMenu方法：

```java
 @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        //定义菜单2：使用ctrl+o进行方法重写，第一个参数是通过哪个资源文件创建菜单，第二个参数就是方法中传入的参数。
        getMenuInflater().inflate(R.menu.main,menu);
        //显示菜单
        return true;
    }
```

对菜单的每个item进行响应-重写onOptionsItemSelected方法：

```java
@Override
    public boolean onOptionsItemSelected(MenuItem item) {
        //定义菜单3：根据item的id进行响应
        switch (item.getItemId()) {
            case R.id.add_item:
                Toast.makeText(this, "点击了新增按钮", Toast.LENGTH_SHORT).show();
                break;
            case R.id.remove_item:
                Toast.makeText(this, "点击了移除按钮", Toast.LENGTH_SHORT).show();
                break;
            default:
        }
        return true;
    }
```
## 活动交互 ##

### 显式intent ###

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    //创建活动2：给活动加载布局，通常情况下这里传入一个布局文件的ID，资源目录下添加的资源都会在R文件中生成一个资源ID
    setContentView(R.layout.first_layout);
    //将取出来的view类型向下转型为Button类型
    final Button button = (Button) findViewById(R.id.button_1);
    button.setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View v) {
       	//隐式组件调用
            Intent intent = new Intent("activityIntent");
            //如果添加了其他category，那么在manifest.xml文件中的intent-filter也要加上该策略
            //intent.addCategory("MY_CATEGORY");
            startActivity(intent);
        }
    });
}
```
### 隐式intent ###

```xml
<activity android:name=".SecondActivity">
<!--指定action的名字，要和activity中的intent的action对应-->
<intent-filter>
    <action android:name="activityIntent"/>
    <category android:name="android.intent.category.DEFAULT"/>
    <!--与activity中的策略对应-->
    <!--<category android:name="MY_CATEGORY"/>-->
</intent-filter>
</activity>
```
活动中的action名字和category名字要和manifest.xml中的一致：

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    //创建活动2：给活动加载布局，通常情况下这里传入一个布局文件的ID，资源目录下添加的资源都会在R文件中生成一个资源ID
    setContentView(R.layout.first_layout);
    //将取出来的view类型向下转型为Button类型
    final Button button = (Button) findViewById(R.id.button_1);
    button.setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View v) {
            //隐式组件调用
            Intent intent = new Intent("activityIntent");
			//如果添加了其他category，那么在manifest.xml文件中的intent-filter也要加上该categoty，android.intent.category.DEFAULT的category会在调用的时候默认添加
            //intent.addCategory("MY_CATEGORY");
            startActivity(intent);
        }
    });
}
```
隐式intent还有其他用法，例如使用一些系统内置action，会调用浏览器打开百度主页：

```java
//使用action_view的action
Intent intent = new Intent(Intent.ACTION_VIEW);
//使用uri.parse方法将一个网址字符串解析为一个uri对象
intent.setData(Uri.parse("http://www.baidu.com"));
startActivity(intent);
```
或者新建自定义活动用来响应http数据协议：

```xml
<!--自定义第三个活动用来响应http数据协议，注意：只有当category包含BROWSABLE类型时，acheme才可以为http-->
        <activity android:name=".ThirdLayoutActivity">
            <intent-filter>
                <action android:name="android.intent.action.VIEW"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <category android:name="android.intent.category.BROWSABLE"/>
                <data android:scheme="http"/>
            </intent-filter>
        </activity>
```
这样一来，当点击按钮的时候在选择列表中就可以看到自己的应用程序，当然它打开的只是第三个活动，而不能显示百度主页。
还可以使用tel协议打开拨号界面：

```java
Intent intent = new Intent(Intent.ACTION_DIAL);
intent.setData(Uri.parse("tel:10086"));
startActivity(intent);
```
### intent传递数据 ###
活动之间通过intent传递数据;

```java
//第一个活动放入数据
String data = "hello second";
Intent intent = new Intent(FirstActivity.this, SecondActivity.class);
intent.putExtra("extra_data", data);
startActivity(intent);
```
取数据：

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    Intent intent = getIntent();
    //可以通过getIntExtra,getStringExtra等取出数据
    String extraData = intent.getStringExtra("extra_data");
    Log.d(TAG, extraData);
    setContentView(R.layout.second_layout);
}
```
### intent返回数据 ###
还可以在第二个活动中给第一个活动返回数据，首先需要修改第一个活动的启动方式：

```java
//第一个活动放入数据
String data = "hello second";
Intent intent = new Intent(FirstActivity.this, SecondActivity.class);
intent.putExtra("extra_data", data);
//使用获取返回值的方式启动其他活动
startActivityForResult(intent,1);
```
第二个活动里放入返回数据：

```java
Intent resultIntent = new Intent();
resultIntent.putExtra("data_return","hello first");
setResult(RESULT_OK, resultIntent);
finish();
```
重写第一个活动的方法来接收返回参数：

```java
@Override
protected void onActivityResult(int requestCode, int resultCode, @Nullable Intent data) {
    //启动活动时传入的请求码
    switch (requestCode){
        case 1:
            //返回数据时的响应码
            Log.d(TAG, data.getStringExtra("data_return"));
            break;
        default:
    }
}
```
可以重写onBackPressed方法来当用户按下返回键的时候返回数据:

```java
@Override
    public void onBackPressed() {
    	Intent resultIntent = new Intent();
		resultIntent.putExtra("data_return","hello first");
		setResult(RESULT_OK, resultIntent);
		finish();   
    }
```