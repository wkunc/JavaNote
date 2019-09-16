# File 类
 虽然叫 File 但是它其实是文件和路径的抽象表示

所以它里面的方法可以分成, 为目录服务的 和 为文件服务的

## 静态方法
FIle 类有三个静态方法, 其中有两个是方法重载
```java
//返回文件系统的根(Root) 在 windows 上有多个, 在 Unix 类系统上只有一个 "/"
//fs 变量是 java.io.FileSystem 的一个实例, 它是一个 abstract 类, fs 实质是一个子类,不同系统不一样
public static File[] listRoots(){
    return fs.listRoot();
}

//用指定的前缀, 后缀 在指定的目录生成一个空的临时文件
public static File createTempFile(String prefix, String suffix, File directory) throws IOException{
    // 前缀长度不能小于3
    if (prefix.length() < 3)
        throw new IllegalArgumentException("Prefix string too short");
    // 后缀没有指定就用 .tmp
    if (suffix == null)
        suffix = ".tmp";
    // 文件目录没有指定就用系统默认临时文件位置
    // TempDirectory 是 File 类中的私有静态内部类, 内部保存了文件系统指定的临时文件位置, location() 就是返回哪个位置
    File tmpdir = (directory != null) ? directory
                                      : TempDirectory.location();
    SecurityManager sm = System.getSecurityManager();
    File f;
    do {
        // 真正生成 FIle 的还是 TempDirectory 中的生产文件方法
        f = TempDirectory.generateFile(prefix, suffix, tmpdir);

        if (sm != null) {
            try {
                sm.checkWrite(f.getPath());
            } catch (SecurityException se) {
                // don't reveal temporary directory location
                if (directory == null)
                    throw new SecurityException("Unable to create temporary file");
                throw se;
            }
        }
    } while ((fs.getBooleanAttributes(f) & FileSystem.BA_EXISTS) != 0);

    if (!fs.createFileExclusively(f.getPath()))
        throw new IOException("Unable to create temporary file");

    return f;
}
//重载了上面的方法,在默认临时文件位置创建空文件, 经测试 win10 下 默认位置在~/AppData/Local/Temp 文件下
public static File createTempFile(String prefix, String suffix) throws IOException{
    retrun createTempFile(prefix, suffix, null);
}
```

## 常见方法
boolean createNewFile()
boolean exists()
String[] list()
String[] list(FilenameFilter filter)
File[] listFiles()
File[] listFiles(FileFilter filter)
File[] listFiles(FilenameFilter filter)

## 注意事项
File 代表的文件或是目录路径是不一定存在, 有些方法是需要你的 File 类实例是存在才能正常工作

比如: 
* long getFreeSpace() // 返回file所在分区的未分配空间

# 18.8 标准I/O
标准I/O这个术语参考的是Unix中 "程序所使用的单一信息流" 这个概念(在Windows和其他许多操作系统中也有相似的实现)

程序的所有输入可以来自**标准输入**, 它的所有输入也可以发送到**标准输出**,已经所有错误可以发送到**标准错误**

标准I/O的意义在于: 我们可以很容易地把程序串联起来, 一个程序的标准输出可以成为另一个程序的标准输入

# 18.8.1 从标准输入中读取
按照标准I/O模型, Java 提供了 System.in, System.out 和 System.err.

System.out 和 System.err 已经被包装成 PrintStream(OutputStream的子类)

只有 System.in 还是一个未经包装的 InputStream, 这意味着我们可以立即使用 System.out 和 System.err 但是在使用
System.in 之前必须对其进行包装

# 18.10 新I/O
jdk1.4 的时候 java.nio.\* 引入了新的 JavaI/O 类库, 其目的在于提高速度, 旧I/O也已经用nio重新实现过了.

速度的提高来自于所使用的结构更接近于操作系统执行I/O的方式: *通道* 和 *缓冲器*

可以把通道看出煤矿, 缓冲器就是矿车, 我们只是和缓冲器交互.

## Buffer
nio 中重要的抽象之一 Buffer 抽象类, 它有多个子类. 最核心的是 ByteBuffer.

唯一直接的和通道交互的缓冲器是 ByteBuffer :可以存储未加工字节的缓冲器.

旧的I/O类库中有三个类被修改了, 用以产生 FileChannel. 这三个类是:
FileInputStream, FileOutputStream, 和 RandomAccessFile.

这些都是字节操作流, 与底层的 nio 性质一致. Reader 和 Writer 这种字符模式类不能用于生产 Channel.
但是 java.nio.channel.Channels 工具类提供了实用方法, 用以在通道中产生 *Reader* 和 *Writer*.


# 内存映射文件
内存映射文件允许我们创建和修改哪些因为太大而不能放入内存的文件.
有了内存映射文件我们就可以完全把它当作非常大的数组来访问.
这种方法大大简化了用于修改文件的代码.


# 文件加锁


