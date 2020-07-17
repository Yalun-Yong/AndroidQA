# JavaIO/NIO, OkIO

## Java 传统 IO

传统 IO 是相较于新出的 IO 来说的。

理解 IO 就要先理解它的抽象，由于传输的数据是一字节一字节传输的，在编程中就被抽象成了一个流，字节像水流一样从程序外部流入程序内部，就是输入流。而由程序内部输入到外部，就是输出流。

```Kotlin
var outputStream: OutputStream? = null 
try {
    outputStream = FileOutputStream("TestIO.txt")
    outputStream.write("hello".toByteArray())
} catch (ex: FileNotFoundException) {
} catch (ex: IOException) {
} finally {
    try {
        outputStream?.close()
    } catch (ex: IOException) {
        ex.printStackTrace()
    }
}


// 1.7 之后，继承了 `Closeable` 接口的类都可以在传给 try 来自动关闭，减少了样板代码。


byte[] bytes = {'i', 'o'};
try ( OutputStream outputStream = new FileOutputStream("TestIO.txt") ){
    outputStream.write(bytes);
} catch (IOException e) {
}

// kotlin try 没有传参数，但是有 use 函数可以自动释放资源

val bytes = byteArrayOf('i'.toByte(), 'o'.toByte())
FileOutputStream("TestIO.txt").use {
        outputStream -> outputStream.write(bytes)
}
```

FileOutputStream 只能读写字节，想要读取字符，就像套接管子一样，需要再接一层。

```
FileInputStream("TestIO.txt").use {
    val word = InputStreamReader(it).read()
    println(word)
}
```

这就构成了如下的结构

FileInputStream --> InputStreamReader --> BufferedReader

> 对于网络

Socket 可以一个 `InputStream` 作为 InputStreamReader 的参数，这样就能套接上原有的类，以一个流的形式输入输出了。这样的设计极大地统一了接口和复用了类。

## flush

输出到缓冲流时，关闭之前，要先 flush，将缓冲区的内容保存到文件。这是因为缓冲为了提高效率，每次写入并不会立即写入到外部文件等，而是积累一批数据后，批量输出。而最后可能剩一些不够一批，所以要最后写出一次。而如果将缓冲对象也放到 try 参数中，也会自动 `flash`。**try 参数可以传入多行代码**。 注意是代码，try 参数传入的是一个代码快。用分好分割多行代码。


## NIO

- 支持非阻塞式的读写
- 使用 Channel 来读写数据
- 双向的，一个类支持输入和输出。
- 强制使用 buffer

## OkIO

1. 也是单向的
2. 也是流的形式
3. 支持非阻塞式
5. 支持缓存但不强制
6. 支持将缓存当做输入输出流

输入是源： Source

```Kotlin 
val file = File("TestIO.txt")
val source = file.source()
    .buffer()
source.use {
    println(source.readUtf8())
}
```

```Kotlin 
val buffer = Buffer()
val writer = buffer.outputStream().bufferedWriter()
writer.write("hhhhhhhhh")
writer.flush()
// 等于 buffer.write("hhhhhhhhh")。本身就是一个缓冲区，没必要再套一层。

println(buffer.readUtf8())
```

## BIO

BIO - Blocking IO 
AIN - Async IO。Java 提供的一套异步的 IO API 可以以会掉的形式编程。