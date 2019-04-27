---
title: Android学习笔记三-界面开发
categories: 学习笔记
date: 2019-04-26 14:26:28
tags: Android
keywords: [Android]
description: 郭霖《第一行代码》之界面开发。
---

# 简介

笔记记载的内容有：常用的几大界面和布局，并仿写一个简单的聊天界面。

<!--more-->

# 布局介绍

## LinearLayout

该布局内元素都是线性排列，如果在<LinearLayout>标签里指定了`android:orientation="vertical"`，则所有元素都是纵向排列的。默认值是horizontial，即横向排列。横向排列时，内部元素的layout_width就不能设置为match_parent，因为该值会使得元素占满一行；由于元素的横向长度不确定，所以此时元素的对齐方式layout_gravity只能为垂直对齐。

对于`android:layout_gravity`和`android:layout`，前者用于指定控件在布局中的对齐方式，后者为控件内的文字等在控件中的对齐方式。

LinearLayout里也有一个重要属性`android:layout_weight`，可以解决手机屏幕的适配问题，LinearLayout会将布局里的所有元素的`anroid:layout_weight`的值相加，然后根据每个元素所占比例和布局的orientation排列进行水平或者垂直切分，如果不设置`anroid:layout_weight`值的控件和使用该属性的控件混合使用，使用wrap_content，则使用wrap_content会单独计算，其余的等比切分。

## RelativeLayout

相对布局内的控件排列比较随意，相对定位顾名思义会有一个参照定位点，例如`android:layout_alignParentTop`，`android:layout_alignParentLeft`等分别表示相对父布局的顶部，相对父布局的右边，或者`android:layout_above=@id/button`，`android:layout_toRightOf=@id/button`分别表示在id为button的的上方，在id为button的右边。

## FrameLayout

帧布局，这种布局默认摆放在布局的左上角。

## PercentLayout

百分比布局是新增的布局，需要在app的build.gradle文件的dependencies闭包下添加`compile 'com.android.support.percent:28.0.0'`，此时在布局文件里就可以使用如下方式来使用了：

```xml
<android.support.percent.PercentFrameLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    <Button
        android:id="@+id/button"
        android:layout_gravity="right|top"
        app:layout_widthPercent="50%"
        app:layout_highPercent="50%"
    />
</android.support.percent.PercentFrameLayout>
```

# 自定义控件

环境：
Android Studio 3.4
Windows 10
调试机：一加 6 Android 9
app/build.gradle如下：   

```groovy
apply plugin: 'com.android.application'

android {
    compileSdkVersion 27

    defaultConfig {
        applicationId "com.joy.practice"
        minSdkVersion 15
        targetSdkVersion 27

        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
        javaCompileOptions { annotationProcessorOptions { includeCompileClasspath = true } }
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
}

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    iimplementation 'com.android.support:appcompat-v7:27.1.1'
    implementation 'com.android.support:recyclerview-v7:27.1.1'

    implementation 'com.android.support.constraint:constraint-layout:1.1.3'
    testImplementation 'junit:junit:4.12'
    androidTestImplementation 'com.android.support.test:runner:1.0.2'
    androidTestImplementation 'com.android.support.test.espresso:espresso-core:3.0.2'
    implementation 'org.projectlombok:lombok:1.18.6'
}
```

模仿ios做一个具有返回和编辑按钮的标题栏，首先新建一个标题栏的布局文件，并在res下新建一个drawable-xhdpi目录，然后将需要的title_bg.png,back_bg.png,edit_bg.png复制进去，布局如下：

```

<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:background="@drawable/title_bg">

    <!--标题栏布局-->
    <Button
        android:id="@+id/title_back"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center"
        android:layout_margin="5dp"
        android:text="back"
        android:textColor="#fff"
        android:background="@drawable/back_bg"

        />

    <TextView
        android:id="@+id/title_text"
        android:layout_width="0dp"
        android:layout_weight="1"
        android:layout_height="wrap_content"
        android:layout_gravity="center"
        android:gravity="center"
        android:text="title"
        android:textColor="#fff"
        android:textSize="24sp"
        />

    <Button
        android:id="@+id/title_edit"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center"
        android:layout_margin="5dp"
        android:text="edit"
        android:textColor="#fff"
        android:background="@drawable/edit_bg"
        />

</LinearLayout>
```

dp一般用来表示空间大小和间距。然后新建一个类去实现标题栏的一些点击功能：

```java
public class TitleLayout extends LinearLayout {

    public TitleLayout(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
        //动态加载标题栏布局 第一个参数是要加载的布局，第二个是root，即父布局
        LayoutInflater.from(context).inflate(R.layout.title, this);
        Button backButton = findViewById(R.id.title_back);
        Button editButton = findViewById(R.id.title_edit);
        backButton.setOnClickListener(view -> {
            ((Activity) getContext()).finish();
        });
        editButton.setOnClickListener(view -> {
            Toast.makeText(getContext(), "点击了编辑按钮", Toast.LENGTH_SHORT).show();
        });
    }
}
```

最后需要在主活动里引入自定义布局：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">
   <!--使用引入布局文件的方式引入-->
   <!--<include layout="@layout/title" />-->
   <!--使用自定义控件方式引入-->
   <com.joyinclude.TitleLayout
       android:layout_width="match_parent"
       android:layout_height="wrap_content">
   </com.joyinclude.TitleLayout>


</LinearLayout>
```

这样就实现了返回键销毁当前Activity，编辑键能够打印文字。

# 列表

## ListView

首先把需要展示的水果图复制到drawable-xhdpi目录里，并创建listview的布局文件activity_list_view.xml：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

   <com.joyinclude.TitleLayout
       android:layout_width="match_parent"
       android:layout_height="wrap_content">
   </com.joyinclude.TitleLayout>

   <!--listview的简单用法-->
   <ListView
       android:id="@+id/list_view"
       android:layout_width="match_parent"
       android:layout_height="wrap_content">
   </ListView>


</LinearLayout>
```

新建一个实体类分别表示水果名和图片id：

```java
public class Fruit {

    private String name;

    private int imageId;

    public Fruit(String name, int imageId) {
        this.name = name;
        this.imageId = imageId;
    }
    //get和set方法省略
}
```

添加一个适配器用于进行水果名和图片的展示：

```java
public class FruitAdapter extends ArrayAdapter<Fruit> {

    private int resourceId;

    class ViewHolder{
        private ImageView imageView;
        private TextView nameView;

    }

    public FruitAdapter(@NonNull Context context, int textViewResourceId, @NonNull List<Fruit> objects) {
        super(context, textViewResourceId, objects);
        resourceId = textViewResourceId;
    }


    /**
     * 该方法在每一个子项加载到屏幕内时使用
     * @param position
     * @param convertView
     * @param parent
     * @return
     */
    @NonNull
    @Override
    public View getView(int position, @Nullable View convertView, @NonNull ViewGroup parent) {
        //获取当前item实例
        Fruit fruit = getItem(position);
        View view;
        ViewHolder viewHolder;
        //重用view，避免每次加载子项时都重复加载布局
        if(convertView == null){
            view = LayoutInflater.from(getContext()).inflate(resourceId, parent, false);
            viewHolder = new ViewHolder();
            viewHolder.imageView = view.findViewById(R.id.fruit_image);
            viewHolder.nameView = view.findViewById(R.id.fruid_name);
            //将viewHolder存在view中
            view.setTag(viewHolder);
        }else{
            view = convertView;
            viewHolder = (ViewHolder) view.getTag();
        }
        //使用内部类持有imageview和textview，放在view中，避免每次重新findviewbyid
        //ImageView fruitImage = view.findViewById(R.id.fruit_image);
        viewHolder.imageView.setImageResource(fruit.getImageId());
        //TextView fruitName = view.findViewById(R.id.fruid_name);
        viewHolder.nameView.setText(fruit.getName());
        return view;
    }
}
```

界面展示代码如下：

```java
public class ListViewActivity extends AppCompatActivity {

    private String[] data = {"Apple","Banana","Orange","Watermelon",
            "pear","Grape","Pineapple","Strawberry","Cherry","Mongo",
            "Apple","Banana","Orange","Watermelon",
            "pear","Grape","Pineapple","Strawberry","Cherry","Mongo"};

    private List<Fruit> fruitList = new ArrayList<>();

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ActionBar actionBar = getSupportActionBar();
        if (actionBar != null) {
            actionBar.hide();
        }
        setContentView(R.layout.activity_list_view);
//        ArrayAdapter<String> adapter = new ArrayAdapter<>(ListViewActivity.this, android.R.layout.simple_list_item_1, data);
        ListView listView = findViewById(R.id.list_view);
        initFruitList();
        FruitAdapter adapter = new FruitAdapter(ListViewActivity.this, R.layout.list_item, fruitList);
        listView.setAdapter(adapter);
        listView.setOnItemClickListener(((adapterView,view,i,l) -> {
            Fruit fruit = fruitList.get(i);
            Toast.makeText(ListViewActivity.this, fruit.getName(), Toast.LENGTH_SHORT).show();
        }));
    }

    private void initFruitList(){
        for (int i = 0; i < 2; i++) {
            Fruit apple = new Fruit("Apple",R.drawable.apple_pic);
            fruitList.add(apple);
            Fruit banana = new Fruit("Banana",R.drawable.banana_pic);
            fruitList.add(banana);
            Fruit orange = new Fruit("Orange",R.drawable.orange_pic);
            fruitList.add(orange);
            Fruit watermelon = new Fruit("Watermelon",R.drawable.watermelon_pic);
            fruitList.add(watermelon);
            Fruit pear = new Fruit("Pear",R.drawable.pear_pic);
            fruitList.add(pear);
            Fruit grape = new Fruit("Grape",R.drawable.grape_pic);
            fruitList.add(grape);
            Fruit pineapple = new Fruit("Pineapple",R.drawable.pineapple_pic);
            fruitList.add(pineapple);
            Fruit strawberry = new Fruit("Strawberry",R.drawable.strawberry_pic);
            fruitList.add(strawberry);
            Fruit cherry = new Fruit("Cherry",R.drawable.cherry_pic);
            fruitList.add(cherry);
            Fruit mongo = new Fruit("Mongo",R.drawable.mango_pic);
            fruitList.add(mongo);
        }
    }
}
```

效果如图：![Android学习笔记三-界面开发\20190426174218](Android学习笔记三-界面开发\20190426174218.jpg)

## RecyclerView

RecyclerView是一个效率更高，并且功能更加强大的滚动控件，需要在app/build.gradle的dependencies闭包里添加支持：

```groovy
implementation 'com.android.support:recyclerview-v7:27.1.1'
```

> 注意：在API 26以后，build.gradle依赖里的compile改成了implementation，testCompile改成了testImplementation。同时，如果想要使用lombok的时候，android闭包下的defaultConfig里还需要加入
> 
> ```groovy
> javaCompileOptions { annotationProcessorOptions { includeCompileClasspath = true } }
> ```

首先创建一个简单的纵向布局activity_cycler_view.xml：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

   <com.joyinclude.TitleLayout
       android:layout_width="match_parent"
       android:layout_height="wrap_content">
   </com.joyinclude.TitleLayout>

   <android.support.v7.widget.RecyclerView
       android:id="@+id/cycler_view"
       android:layout_width="match_parent"
       android:layout_height="match_parent"></android.support.v7.widget.RecyclerView>

</LinearLayout>
```

并创建与之对应的活动：

```java
public class CyclerViewActitivy extends AppCompatActivity {

    private List<Fruit> fruitList = new ArrayList<>();

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_cycler_view);
        ActionBar actionBar = getSupportActionBar();
        if (actionBar != null) actionBar.hide();
        initFruitList();
        RecyclerView recyclerView = findViewById(R.id.cycler_view);
        LinearLayoutManager linearLayoutManager = new LinearLayoutManager(this);
        recyclerView.setLayoutManager(linearLayoutManager);
        FruitCyclerAdapter fruitCyclerAdapter = new FruitCyclerAdapter(fruitList);
        recyclerView.setAdapter(fruitCyclerAdapter);
    }

    private void initFruitList(){
        for (int i = 0; i < 2; i++) {
            Fruit apple = new Fruit("Apple",R.drawable.apple_pic);
            fruitList.add(apple);
            Fruit banana = new Fruit("Banana",R.drawable.banana_pic);
            fruitList.add(banana);
            Fruit orange = new Fruit("Orange",R.drawable.orange_pic);
            fruitList.add(orange);
            Fruit watermelon = new Fruit("Watermelon",R.drawable.watermelon_pic);
            fruitList.add(watermelon);
            Fruit pear = new Fruit("Pear",R.drawable.pear_pic);
            fruitList.add(pear);
            Fruit grape = new Fruit("Grape",R.drawable.grape_pic);
            fruitList.add(grape);
            Fruit pineapple = new Fruit("Pineapple",R.drawable.pineapple_pic);
            fruitList.add(pineapple);
            Fruit strawberry = new Fruit("Strawberry",R.drawable.strawberry_pic);
            fruitList.add(strawberry);
            Fruit cherry = new Fruit("Cherry",R.drawable.cherry_pic);
            fruitList.add(cherry);
            Fruit mongo = new Fruit("Mongo",R.drawable.mango_pic);
            fruitList.add(mongo);
        }
    }
}
```

创建RecyclerView中每一行数据中，文字和图片的布局为水平：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content">

    <!--垂直滚动的布局-->
    <ImageView
        android:id="@+id/fruit_image"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" />

   <TextView
       android:id="@+id/fruit_name"

       android:layout_width="wrap_content"
       android:layout_height="wrap_content"
       android:layout_gravity="center_vertical"
       android:layout_marginLeft="10dp"/>

</LinearLayout>
```

并且创建对应的数据适配器：

```java
public class FruitCyclerAdapter extends RecyclerView.Adapter<FruitCyclerAdapter.ViewHolder> {

    private List<Fruit> fruitList;

    @NonNull
    @Override
    public ViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        //传统纵向滚动布局
        View view = LayoutInflater.from(parent.getContext()).inflate(R.layout.list_item, parent, false);

        //创建viewHolder，将父view传入并对子项每个元素绑定点击事件
        ViewHolder viewHolder = new ViewHolder(view);
        viewHolder.imageView.setOnClickListener(event -> {
            int position = viewHolder.getAdapterPosition();
            Fruit fruit = fruitList.get(position);
            Toast.makeText(view.getContext(), "iamge:"+fruit.getName(), Toast.LENGTH_SHORT).show();
        });
        viewHolder.textView.setOnClickListener(event -> {
            int posotion = viewHolder.getAdapterPosition();
            Fruit fruit = fruitList.get(posotion);
            Toast.makeText(view.getContext(), "name:" + fruit.getName(), Toast.LENGTH_SHORT).show();
        });
        return viewHolder;
    }

    @Override
    public void onBindViewHolder(@NonNull ViewHolder holder, int position) {
        //对子项每个元素进行赋值

        Fruit fruit = fruitList.get(position);
        holder.imageView.setImageResource(fruit.getImageId());
        holder.textView.setText(fruit.getName());
    }

    @Override
    public int getItemCount() {
        //告诉父布局一共有多少元素

        return fruitList.size();
    }

    static class ViewHolder extends  RecyclerView.ViewHolder{

        ImageView imageView;

        TextView textView;

        public ViewHolder(View itemView) {
            super(itemView);
            //获取垂直滚动元素
            imageView =  itemView.findViewById(R.id.fruit_image);

            textView =  itemView.findViewById(R.id.fruit_name);

        }
    }

    public FruitCyclerAdapter(List<Fruit> fruitList) {
        this.fruitList = fruitList;
    }
}
```

运行软件效果如图：

![Android学习笔记三-界面开发\20190426174516](Android学习笔记三-界面开发\20190426174516.jpg)

点击图片和文字能够输出不同的文字，ListView实现方式则是直接注册的OnItemClickListener，无法对每一个item里的每一个元素做出响应，虽然也可以实现，但是比较复杂。相比之下RecylerView对于每一个item的操作都是注册到view上的。

RecyclerView还可以实现横向滚动的列表，此时每一个item里面的元素就是垂直摆放了，调整list_item.xml的代码：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="100dp"
    android:layout_height="wrap_content">

    <!--横向滚动的布局-->
    <ImageView
        android:id="@+id/fruit_image"

        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center_horizontal"
        />

   <TextView
       android:id="@+id/fruit_name"

       android:layout_width="wrap_content"
       android:layout_height="wrap_content"
       android:layout_gravity="center_horizontal"
       android:layout_marginTop="10dp"/>


</LinearLayout>
```

使用center_vertical将图片和文字都水平居中，并且使用layout_marginTop让文字和图片之间保持10dp的间距。然后修改CyclerViewActivity里的代码，调整LinearLayoutManager里的orientation属性为水平：

```java
        LinearLayoutManager linearLayoutManager = new LinearLayoutManager(this);
        //设置横向滚动
        linearLayoutManager.setOrientation(LinearLayoutManager.HORIZONTAL);
        recyclerView.setLayoutManager(linearLayoutManager);
```

效果如下：

![Android学习笔记三-界面开发\20190426181814](Android学习笔记三-界面开发\20190426181814.jpg)

还可以实现瀑布滚动效果，首先调整list_item.xml：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_margin="5dp"
    >

    <!--瀑布流布局-->
    <ImageView
        android:id="@+id/fruit_image"

        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center_horizontal"
        />

   <TextView
       android:id="@+id/fruit_name"

       android:layout_width="wrap_content"
       android:layout_height="wrap_content"
       android:layout_gravity="left"
       android:layout_marginTop="10dp"/>

</LinearLayout>
```

每一项布局设定为match_parent用于自动适配，layout_margin保证每个水果之间有一定间距，文本用于居左对齐，因为待会要进行名称改造，名称比较长时就能看到瀑布的效果了。

修改活动类里的代码：

```java
        RecyclerView recyclerView = findViewById(R.id.cycler_view);
        StaggeredGridLayoutManager staggeredGridLayoutManager = new StaggeredGridLayoutManager(3, StaggeredGridLayoutManager.VERTICAL);
        recyclerView.setLayoutManager(staggeredGridLayoutManager);
```

初始化数据时生成随机长度水果名称:

```java
    private void initFruitList(){
        for (int i = 0; i < 2; i++) {
            Fruit apple = new Fruit(getRandomLengthName("Apple"),R.drawable.apple_pic);
            fruitList.add(apple);
            Fruit banana = new Fruit(getRandomLengthName("Banana"),R.drawable.banana_pic);
            fruitList.add(banana);
            Fruit orange = new Fruit(getRandomLengthName("Orange"),R.drawable.orange_pic);
            fruitList.add(orange);
            Fruit watermelon = new Fruit(getRandomLengthName("Watermelon"),R.drawable.watermelon_pic);
            fruitList.add(watermelon);
            Fruit pear = new Fruit(getRandomLengthName("Pear"),R.drawable.pear_pic);
            fruitList.add(pear);
            Fruit grape = new Fruit(getRandomLengthName("Grape"),R.drawable.grape_pic);
            fruitList.add(grape);
            Fruit pineapple = new Fruit(getRandomLengthName("Pineapple"),R.drawable.pineapple_pic);
            fruitList.add(pineapple);
            Fruit strawberry = new Fruit(getRandomLengthName("Strawberry"),R.drawable.strawberry_pic);
            fruitList.add(strawberry);
            Fruit cherry = new Fruit(getRandomLengthName("Cherry"),R.drawable.cherry_pic);
            fruitList.add(cherry);
            Fruit mongo = new Fruit(getRandomLengthName("Mongo"),R.drawable.mango_pic);
            fruitList.add(mongo);
        }
    }

    private String getRandomLengthName(String name){
        Random random = new Random();
        int length = random.nextInt(20) + 1;
        StringBuilder stringBuilder = new StringBuilder();
        for (int i = 0; i < length; i++) {
            stringBuilder.append(name);
        }
        return stringBuilder.toString();
    }
```

效果如下：

![Android学习笔记三-界面开发\20190426182816](Android学习笔记三-界面开发\20190426182816.jpg)

# 界面最佳实践

环境：
Android Studio 3.4
Windows 10
调试机：一加 6 Android 9
创建一个项目，其中app/build.gradle如下：

```groovy
apply plugin: 'com.android.application'

android {
    compileSdkVersion 28
    defaultConfig {
        applicationId "com.joy.practice"
        minSdkVersion 15
        targetSdkVersion 28
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
}

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'com.android.support:appcompat-v7:28.0.0'
    implementation 'com.android.support:recyclerview-v7:28.0.0'
    implementation 'com.android.support.constraint:constraint-layout:1.1.3'
    testImplementation 'junit:junit:4.12'
    androidTestImplementation 'com.android.support.test:runner:1.0.2'
    androidTestImplementation 'com.android.support.test.espresso:espresso-core:3.0.2'
}
```

在android studio里，可以对某个png右击，然后点击creat 9-patch file，即可以对图片进行定制，例如聊天气泡是可以随着文字的增多而进行拉伸的，如图：

![Android学习笔记三-界面开发\Snipaste_2019-04-27_01-44-01](Android学习笔记三-界面开发\Snipaste_2019-04-27_01-44-01.png)

其中四周的黑色边即代表需要拉伸的部分，然后将该图片放到res/drawable-xhdpi目录里。

创建聊天界面布局：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="#d8e0e8">

    <com.joy.practice.TitleLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content">
    </com.joy.practice.TitleLayout>

    <android.support.v7.widget.RecyclerView
        android:id="@+id/msg_view"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"></android.support.v7.widget.RecyclerView>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content">

        <EditText
            android:id="@+id/input_text"
            android:layout_weight="1"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:hint="type something here"
            android:maxLines="2"
            />

        <Button
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Send"
            android:textAllCaps="false"
            android:id="@+id/send"
            />
    </LinearLayout>

</LinearLayout>
```

每一条聊天消息的布局：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:padding="10dp">

    <LinearLayout
        android:id="@+id/left_layout"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="left"
        android:background="@drawable/message_left">

        <TextView
            android:id="@+id/left_msg"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="center"
            android:layout_margin="10dp"
            android:textColor="#fff" />
    </LinearLayout>

    <LinearLayout
        android:id="@+id/right_layout"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="right"
        android:background="@drawable/message_right">

        <TextView
            android:id="@+id/right_msg"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_margin="10dp"
            />
    </LinearLayout>
</LinearLayout>
```

其中title-layout就是上节中有回退和编辑按钮的标题栏，新建一个基类活动用于隐藏系统自带标题栏，以后其他类都可以继承这个基类。

```java
public class BaseActivity extends AppCompatActivity {

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ActionBar actionBar = getSupportActionBar();
        if(actionBar != null) actionBar.hide();
    }
}
```

消息实体类：

```java
public class Msg {

    public static final int TYPE_RECEIVED = 0;

    public static final int TYPE_SENT = 1;

    private String content;

    private int type;

    public Msg(String content, int type) {
        this.content = content;
        this.type = type;
    }

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }

    public int getType() {
        return type;
    }

    public void setType(int type) {
        this.type = type;
    }
}
```

创建工具类判断是否是空字符串：

```java
public class StringUtils {

    public static boolean isEmpty(String str){
        return str == null || str.length() == 0;
    }

}
```

创建一个主活动：

```java
public class MainActivity extends BaseActivity {

    private List<Msg> msgList = new ArrayList<>();

    private EditText inputText;

    private Button send;

    private RecyclerView msgRecyclerView;

    private MsgAdapter msgAdapter;

    @Override

    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        initMsg();
        inputText = findViewById(R.id.input_text);
        send = findViewById(R.id.send);
        msgRecyclerView = findViewById(R.id.msg_view);
        LinearLayoutManager layoutManager = new LinearLayoutManager(this);
        msgRecyclerView.setLayoutManager(layoutManager);
        msgAdapter = new MsgAdapter(msgList);
        msgRecyclerView.setAdapter(msgAdapter);
        send.setOnClickListener(view -> {
            String content = inputText.getText().toString();
            if (StringUtils.isEmpty(content)) {
                Toast.makeText(msgRecyclerView.getContext(), "不可发送空白消息", Toast.LENGTH_SHORT).show();
            } else {
                Msg msg = new Msg(content, Msg.TYPE_SENT);
                msgList.add(msg);
                //发送消息插入通知

                msgAdapter.notifyItemInserted(msgList.size() - 1);
                //将新元素插入到队尾

                msgRecyclerView.scrollToPosition(msgList.size() - 1);
                inputText.setText("");
            }
        });
    }

    private void initMsg() {
        Msg msg1 = new Msg("我爱萌儿", Msg.TYPE_SENT);
        msgList.add(msg1);
        Msg msg2 = new Msg("我也爱你", Msg.TYPE_RECEIVED);
        msgList.add(msg2);
        Msg msg3 = new Msg("我想娶你", Msg.TYPE_SENT);
        msgList.add(msg3);
    }
}
```

效果如图：


