## 1. Bitmaip 和　Ｄrawable 区别

Bitmap 是由像素值组成的，是图片资源在内存中位图信息的存储。

Drawable 是类似 View　的的绘制工具，可以调用 Canvas 等工具绘制图形。只有绘制功能，没有事件处理机制。

互转：　所谓互转，并不是转换，其实是由一个类根据在屏幕显示的规则生成另一个类。

- Bitmap 转 Drawable，由于 Drawable 是一个绘制工具，它可以绘制一个图片，新生成一个　BitmapDrawable, 将 Bitmap 作为一个内部变量存贮来绘制。
- Drawable 转　Bitmap 如果是 BitmapDrawabl 只需要取出其中的 Bitmap, 如果是普通的 Drawable, 则需要计算绘制时的每个像素值，填到 Bitmap 中。

## Drawable 和　View 的区别

1. View 绘制包括测量、布局、绘制，而 Drawable　只包含绘制。

2. Drawable 使用时需要设置边界，然后调用 draw() 方法即可。

## asset 目录和 res/raw 目录的区别

相同点：两者目录下的文件在打包后都会原封不动的报讯在　apk 包中，不会编译成二进制。

不同点：

1. res/raw 的文件会被映射到　R.java 文件中，访问的时候可以直接通过　R.id　访问资源id. assets　目录下的资源不会被映射到 R.java，访问的时候需要通过　AssetManager 类通过路径访问。
2. res/raw 不可以有目录，而　assets 可以建目录结构。

## 组件化

组件化其实就是模块化，把软件按照功能，相同的功能拆分成不同的`module`，来方便开发软件。

## 插件化

好处：动态部署，添加新功能。

不是它的优点的：

- 多 app 联合打包。并不是插件化的优点，使用判断打包是否打包成一个包就能实现，或者 AAB 功能来实现。插件化太过繁琐。
- 解耦。解耦重在设计，如果设计的好，多 module 就能实现解耦。如果开发设计不够规范，多 app 的耦合依旧会很高。 

插件是拆分出来的模块，在打包的时候不打包在 apk 中，而是留一个接口，在用户使用的时候，根据需要，能够自动下载添加扩展功能。一种方案是把模块打包成一个 apk，让用户需要的时候下载 apk，主软件在动态的读取模块　apk.

实现的方法就是使用 `ClassLoader` 自己加载类文件，然后反射执行方法。

### 方法一

1. 将插件打包成　apk 方法放到 asstes 目录下

2. 因为 `assets` 目录下的文件不能任意解压，所以可以将文件解压到　cash 目录，然后通过 `DexClassLoader` 加载类。

3. 通过反射来获取加载类的方法。

存在的问题:
    
1. 无法启动另一个包里的　Activity，因为启动 Activity 都要在 Manifest 中注册，不注册不能启动。解决办法

    1. 使用代理　Activity 来执行加载的插件的　Activity 的方法。
    2. 重写打包过程，把两个包的　Manifest 打包到一起。这样显然不可以，之所以要插件化，就是为了动态添加。打包到一起就还是不能动态添加新功能了。
    3. 欺骗系统，系统启动 Activity 的过程是先检测要启动的，然后再启动。可以在系统检测的之前，提交的是启动本身已有的 Activity，等到验证过之后，启动的实际是另一个 Activity。这个问题也显而易见，安卓系统厂商发现这个问题也会屏蔽。
2. 无法获取资源文件。解决办法：
    1. 重写 `Activity` 的 `getResources` 和 `getAssets`方法，



## AAB 

Android App Bundle 是 google 提供的包拆分功能，能够根据用户手机的配置，仅打包用户需要的资源，例如多语言，不同尺寸的图片资源等。从而减少包体积。另一个功能是动态组件(Dynamic Feature)，将换一个功能单独放到一个 module，根据配置，能够在用户需要的时候，再动态的加载一个组件。


## 热更新或热修复

不像插件化，一是热更新是原先不知道要更新或修复的是哪一部分内容，任何地方都可能出现问题。二是插件化会写代码留一个入口，用户通往新功能。而热修复是对原有代码修改，并没有什么入口代码。

热更新有许多方法，其中一种方法是 `ClassLoader`,


## RecyclerView

### 和 ListView 的对比

1. 只支持垂直的滑动，对于横向滑动没有直接支持。使用列表布局，流布局则需要使用其他类。
2. 没有支持动画的 API
3. 接口设计和其他 View 设计不一致，其他View 都是在本身设置点击事件，而它需要设置　`onItemClickListener`等，容易产生混乱。
4. 没有强制实现 ViewHolder, 对于缓存的使用完全凭借开发者的经验，性能比较差。
5. 性能不如 RecyclerView。提供了 DiffUtils 等页面数据更新工具，是数据的部分修改更加高效和平滑。
6. (对于列表中填充不同数据类型没有直接的支持，实现比较麻烦。)

### ViewHolder 是什么？

解决的是什么问题？

ViewHolder 用于保存 item view 中子 view 的引用，减少复用组件的时候　`findViewById` 的性能损耗。

和 item view 的对应关系？

ViewHolder 和　view item 是一对一关系

ListView 中的 ViewHolder 和 item 的复用有什么关系？

item 的复用由用户判断，返回是否为空，为空需要从新创建，不空则复用。而 ViewHolder 仅仅是在不为空复用时，减少 findViewById 的性能损耗。

### 缓存比较

ListView:

RecyclerBin 的类管理缓存复用。ListView 每次渲染的时候，要把所有的绘制内容清空掉，然后根据最近的状态从新计算、绑定所有 View 的数据，然后进行渲染。这样很耗时，因为绝大多数，除了位置不同，其他内容都是不变的。这样计算性能很低。

1. Active View: 屏幕里面能看到的 View.
2. Scrap View: 移除屏幕的 View。


Recycler View 缓存机制

1. Scrap: 屏幕内的 ViewHolder。数据是干净的，不会从新绑定数据，只需要计算位置，就可以拿来绘制。
2. Cache：滑出屏幕的 ViewHolder。当用户反过来滑动的时候，数据也是干净的，可以根据　position 获取到，拿过来直接用。也不用绑定数据。
3. ViewCacheExtension：用户自定义的缓存策略。一个可能的应用场景是，这个可以用户多个 List　中使用一个　View。而不用再次创建了。cache 只关心　position，不关系 ViewType。RecycledViewPool 只关心　ViewType，都需要从新绑定数据。要想实现长时间滑动出去，还是能不用绑定数据，可以在此自定义缓存策略。
4. RecycledViewPool：废弃的 View,数据是脏的，有可能是其他 item 的数据，需要从新绑定数据。多个列表可以设置复用一个，从而减少 View　的创建。

> 1. 展示次数统计

`onBindViewHolder` 可能复用而不再次绑定，需要使用 `onViewAttachedWindow` 来统计。

### RecyclerView 性能优化

1. 在 onCreateView 中绑定监听器。通过　`getAdapterPosition` 获取位置来处理事件。

2. 设置 `LinearLayoutManager.setInitialPrefetchCount`　来设置提前加载数据的个数来避免比较复杂的 item 创建的耗时，提高滑动的流畅性。只有设置嵌套内部的　LayoutManager 才有用，外部的列表的 Manager 的设置是没有用的。

3. setFixedSize(), 当设置为 true 时，当内容变化的时候（添加、删除、更改）,会直接重新子 View，而不会从新布局自身。当 数据改变的时候，不会影响RecyclerView 的大小改变，就可以设置为 true。

4. 多个 List 的 item 是一样的，例如多个 Tab 下的列表，item 是一样的。设置  `setRecycleViewPool` 

5. 使用 DiffUtil 来更新数据，提高效率。不用更新整个列表，同时还有动画，比较友好。当数据量很大的时候，还是需要将计算过程放到后台线程，然后将结果返回到前台线程更新 UI。或者使用 Google 封装好的 `AsyncListDiffer(Executor)/ListAdapter` 来处理。

### ItemDecoration 作用。

1.分割线，2. 蒙版，3.分组间隔。可以设置多个 ItemDecoration，来分别负责不同的效果。实现了解耦。



onDraw 在 imem 之前绘制，会在 Item 的下面。
onDrawOver 在 item 之后绘制，会绘制在 Item 上面。
getItemOffSets Item 的偏移量，用来留出 Item 之间的距离。


更多使用

https://github.com/h6ah4i/android-advancedrecyclerview
https://advancedrecyclerview.h6ah4i.com/










