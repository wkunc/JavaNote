# VFS

VFS 表示虚拟文件系统(Virtual File System), 它用来查找指定路径下的资源.
VFS 是一个抽象类, MyBatis 中提供了JBoss6VFS 和 DefaultVFS 两个实现.

VFS 的核心字段的含义如下:

```java
public static final Class<?>[] IMPLEMENTATIONS = { JBoss6VFS.class, DefaultVFS.class };

public static final List<Class<? extends VFS>> USER_IMPLEMENTATIONS = new ArrayList<Class<? extends VFS>>();
```

```java
// 私有内部类,
  private static class VFSHolder {
    static final VFS INSTANCE = createVFS();

    @SuppressWarnings("unchecked")
    static VFS createVFS() {
      // Try the user implementations first, then the built-ins
      List<Class<? extends VFS>> impls = new ArrayList<Class<? extends VFS>>();
      impls.addAll(USER_IMPLEMENTATIONS);
      impls.addAll(Arrays.asList((Class<? extends VFS>[]) IMPLEMENTATIONS));

      // Try each implementation class until a valid one is found
      VFS vfs = null;
      for (int i = 0; vfs == null || !vfs.isValid(); i++) {
        Class<? extends VFS> impl = impls.get(i);
        try {
          vfs = impl.newInstance();
          if (vfs == null || !vfs.isValid()) {
            if (log.isDebugEnabled()) {
              log.debug("VFS implementation " + impl.getName() +
                  " is not valid in this environment.");
            }
          }
        } catch (InstantiationException e) {
          log.error("Failed to instantiate " + impl, e);
          return null;
        } catch (IllegalAccessException e) {
          log.error("Failed to instantiate " + impl, e);
          return null;
        }
      }

      if (log.isDebugEnabled()) {
        log.debug("Using VFS adapter " + vfs.getClass().getName());
      }

      return vfs;
    }
  }
```

VFS 中定义了 list(URL, String) 和 isValid() 两个抽象方法.
isValid() 负责检测当前 VFS 对象在当前环境是否有效.
list(URL, String) 方法负责查找指定的资源名称列表.
在 ResolverUtil.find() 方法查找类文件时会调用 list() 方法的重载,
该重载最终会调用 list(URL, String).

DefaultVFS.list(URL, String)方法实现如下:

```java

```
