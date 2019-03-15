---
title: Android学习笔记二-Activity
categories: 编程技术
date: 2019-03-13 11:07:00
tags:
- Android
keywords: [Android]
description: 学习《第一行代码》所记笔记
---
# 生命周期
## 返回栈
活动都是存放在返回栈（back stack）里，每个返回栈称之为一个任务（Task）。系统总是显示栈顶的活动（Activity）给用户。

<!--more-->

## 生命周期

![](Android学习笔记二-Activity/18364230.jpg)

1. onCreate():组件的创建，加载布局，绑定事件；
2. onStart():活动即将变为可见；
3. onResume():活动处于交互状态，栈顶；
4. onPause():活动释放一些资源和CPU占用，或者保存关键数据；
5. onStop():活动转变为不可变状态；
6. onDestroy():活动销毁之前调用，标记活动为销毁状态方便系统回收。
7. onRestart():活动由不可见状态变为可见状态的过程。

上面的状态可以分为三大类：

1. 完整生存期：onCreate()-onDestroy()
2. 可见生存期：onStart()-onStop()
3. 前台生存期：onResume()-onPause()

## 代码
新建一个android studio项目，除了主活动MainActivity之外，还有其他两个活动，分别命名为NormalActivity和DialogActivity。

其中normal的布局为：

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent">


    <TextView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="this normal activity"
        />

</LinearLayout>
```
dialog的布局为：

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="this is a dialog activity"
        />

</LinearLayout>
```
然后在主活动main里面添加两个按钮，通过点击按钮来观察生命周期的变化。第一个按钮会停止当前的活动并跳转到一个新的活动，第二个按钮会在当前活动上层叠一个dialog，当前活动会暂停但是不会停止。

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <Button
        android:id="@+id/button1"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="start normal activity"
        />

    <Button
        android:id="@+id/button2"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="start dialog activity"
        />


</LinearLayout>
```
normal和dialog的activity里保持默认即可，在主活动里添加对按钮的响应事件：

```java
public class MainlActivity extends AppCompatActivity {

    private static final String TAG = "MainlActivity";
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        Log.i(TAG, "onCreate: ");
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Button startButton = findViewById(R.id.button1);
        Button dialogButton = findViewById(R.id.button2);
        startButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Intent intent = new Intent(MainlActivity.this,NormalActivity.class);
                startActivity(intent);
            }
        });

        dialogButton.setOnClickListener(view -> {
            Intent intent = new Intent(MainlActivity.this, DialogActivity.class);
            startActivity(intent);
        });
    }

    @Override
    protected void onDestroy() {
        Log.i(TAG, "onDestroy: ");
        super.onDestroy();
    }
    //...分别重写onStart(),onPause()等其他生命周期方法并打印日志
}
```
> 小技巧：如果想要使用JDK 8的lambda表达式，只需要在app目录下的build.gradle里android闭包里添加如下代码即可：
> 
	compileOptions {
	        sourceCompatibility 1.8
	        targetCompatibility 1.8
	    }

执行代码，观察结果：

启动时可以看到：

	I/MainlActivity: onCreate: 
	I/MainlActivity: onStart: 
	I/MainlActivity: onResume:

点击start normal activity时，控制台打印结果：

	I/MainlActivity: onPause: 
	I/MainlActivity: onStop: 

点击返回键：

	I/MainlActivity: onRestart: 
	I/MainlActivity: onStart: 
	I/MainlActivity: onResume: 

点击start dialog activity：

	I/MainlActivity: onPause: 

点击返回键：

	I/MainlActivity: onResume: 
	
在主活动页面点击返回：

	onPause:
	onStop:
	onDestroy:
	
## 数据临时保存
当上一个活动进入stop的生命周期时，此时由于系统内存的不足，可能会导致上一个活动被回收，假如在上一个活动里输入了一些关键数据，那此时再回到上一个活动经历的又是一个create-start-resume的过程，用户还需要重新输入一遍数据，这种用户体验无疑是很差的。这里可以将数据临时保存,onSaveInstanceState方法能够保证系统回收之前一定被执行：

```java
@Override
protected void onSaveInstanceState(Bundle outState) {
    super.onSaveInstanceState(outState);
    outState.putString("tmpString", "last String");
}
```
然后在onCreate中再重新取出来：

```java
if (savedInstanceState != null) {
    Log.i(TAG, "onCreate: "+savedInstanceState.getString("tmpString"));
}
```
# 启动模式
启动模式的配置方式为：AndroidManifest.xml文件中的activity标签内：

```xml
<activity android:name=".ThirdLayoutActivity"
            android:launchMode="singleInstance">
```
活动有四种启动模式：

* standard:每次启动一个活动都会新建一个活动，例如我在main活动上又启动了一个main活动，记做main2，又启动一个记做main3，那么返回栈从上到下依次是：main3-main2-main，同理返回也是一次出栈；
* singleTop:每次启动一个活动时会判断当前返回栈的栈顶是不是要启动的活动，如果是，则不会新建一个活动，直接使用当前活动；如果不是，则会新建活动；
* singleTask:每次启动一个活动会判断当前栈里是否存在需要创建的活动，如果存在，就将该活动压到栈顶成为，原本处于它上方的活动会依次出栈destroy；
* singleInstance:该模式的活动会启动一个新的返回栈来存放，当多个应用需要使用共享的活动时，可以使用该模式。

# 实践
## 继承-妖魔现形
调试的时候或者打印日志的时候，我们通常希望能够知道当前所处的位置或者说类，并且在一个位置做某些通用操作，这里可以通过新建一个BaseActivity类来继承原来代码中的AppCompatActivity，然后让真正的活动来继承我们的BaseActivity从而实现一些统一行为：

```java
public class BaseActivity extends AppCompatActivity {

    private static final String TAG = "BaseActivity";

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Log.d(TAG, getClass().getSimpleName());
    }
}
```
这里只是简单打印一下当前所处的类。

## 封装-天下一统
当我们的activity创建的比较多的时候，如果想一次性退出，比如在某个活动界面放置一个按钮，这个按钮是销毁所有的acivity，可以通过封装一个容器类，收集所有的活动，当用户点击退出软件的按钮，所有的activity都会销毁，避免我们一直按返回。

```java
public class ActivityCollector {

    public static List<Activity> activityList = new ArrayList<>();

    public static void addActivity(Activity activity){
        activityList.add(activity);
    }

    public static void removeActivity(Activity activity) {
        activityList.remove(activity);
    }

    public static void finishAll(){
        activityList.forEach(activity -> {
            if (!activity.isFinishing()) {
                activity.finish();
            }
        });
        activityList.clear();
    }

}
```

然后，在BaseActivity里可以控制创建activity的时候就加入容器类，销毁就从容器类里移除：

```java
public class BaseActivity extends AppCompatActivity {

    private static final String TAG = "BaseActivity";

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Log.d(TAG, getClass().getSimpleName());
        ActivityCollector.addActivity(this);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        ActivityCollector.removeActivity(this);
    }
}
```
然后给退出软件的按钮添加点击事件：

```java
button3.setOnClickListener(view -> {
            ActivityCollector.finishAll();
            android.os.Process.killProcess(android.os.Process.myPid());
        });
```
android.os.Process.killProcess可以杀掉当前进程，myPid即为当前进程号。

## 依赖-控制反转
当我们从活动1调用活动2的时候，可能需要传递一些数据，通常情况下，我们并不知道要传递多少数据以及数据类型，并且需要new一个inten然后放入数据，例如：

```java
String data = "hello second";
Intent intent = new Intent(FirstActivity.this, SecondActivity.class);
intent.putExtra("extra_data", data);
startActivity(intent);
```
但是，如果我们在活动2里就定义好一个方法用来组装数据，那活动1就只需要调用该方法，传入参数即可，而无需手动组装。

SecondActivity.java:

```java
public static void actionStart(Context context,String para1,String para2){
        Intent intent = new Intent(context,SecondActivity.class);
        intent.putExtra("param1", para1);
        intent.putExtra("param2", para2);
        context.startActivity(intent);
}
```
那么在活动1中，只需要`SecondActivity.actionStart(FirstActivity.this,"test","test");`即可跳转到第二个活动。