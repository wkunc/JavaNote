# 2.5 操作文件

## 2.5.1 Path (since 1.7)

Path 表示的是一个目录名序列, 其后还可以跟着一个文件名

```java
//Paths 类中的静态方法, 可以帮助我们获得 Path 对象
public static Path get(String first, String... more){
    return FileSystems.getDefault().getPath(first, more);
}
```

Path 和之前的 File 类是一样的抽象, 只不过 Path 类中的方法更偏向路径的组合解析,
而 File 类中的方法更偏重 File 和 Dir 的控制

Path 类中有很多方法可以让我们方便的获得以原path对象组合别的Path 对象,
或者通过对 Path 对象的解析获得其父路径,子路径等组合解析方法

## 2.5.2 读写文件

Files类可以使普通的文件操作变得快捷

```java
//可以用 readAllBytes 方法, 方便的读取文件所有内容
byte[] bytes = Files.readAllBytes(path);
//可以将文件当前 行(line) 序列(List) 读入
List<String> lines = Files.readAllLines(path, charset);
//向文件输出一个字符串可以用 write 方法
Files.write(path, content.getByte(charset));
//向文件追加内容
Files.write(path, contenet.getByte(charset), StandardOpenOption.APPEDN);
//还可用将 行的集合写入文件
Files.write(path, lines);
```

这些方法适用于处理中等长度的文本文件, 所以要处理的文件长度比较大, 或者是二进制文件,
那么还是应该使用所熟知的 输入/输出流 或者 读入器/写出器

这些方法可以帮助你从FileInputStream, FileOutputStream, BufferedReader 和 BufferWriter 的繁复的操作中解脱出来

# 2.6 内存映射文件

FileChannel channel = FileChannel.open(path, options);

