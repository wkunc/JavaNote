# 2.5 操作文件
## 2.5.1 Path
Path 表示的是一个目录名序列, 其后还可以跟着一个文件名

```java
public static Path get(String first, String... more){
    return FileSystems.getDefault().getPath(first, more);
}
```

Files类可以使普通的文件操作变得快捷 