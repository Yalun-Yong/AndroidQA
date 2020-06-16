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





