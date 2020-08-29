### ClassPool  ver  3.18.1-GA

1. ClassPath接口  

   1. 用来寻找类文件的字节数组
   2. 
   3.  ByteArrayClassPath
   4.  JarClassPath,  JarDirClassPath, DirClassPath

2. ClassPathList   一个链表. 用来表示搜索的路径顺序.   

   1.  insertClassPath(ClassPath cp) 头部插入, 插入到最前
   2.  appendClassPath(ClassPath cp) 遍历后, 插入到最后
   3.  removeClassPath(ClassPath cp) 遍历后, 找到cp后移除(判断 ==)

3. ClassPool  

   1. ClassPool 是 CtClass 对象的容器

   2. ```java
      insertClassPath(String pathname)
      ```

      会根据判断返回不同的ClassPath实现类  

      ```java
      String lower = pathname.toLowerCase();
      if (lower.endsWith(".jar") || lower.endsWith(".zip"))
          return new JarClassPath(pathname);
      
      int len = pathname.length();
      if (len > 2 && pathname.charAt(len - 1) == '*'
          && (pathname.charAt(len - 2) == '/'
              || pathname.charAt(len - 2) == File.separatorChar)) {
          String dir = pathname.substring(0, len - 2);
          return new JarDirClassPath(dir);
      }
      
      return new DirClassPath(pathname);
      ```

   3. **ClassPool 对象用于维护类和 CtClass 对象之间的一对一映射关系**.  为了保证程序的一致性，Javassist 不允许用两个不同的 CtClass 对象来表示同一个类，除非创建了两个独立的 ClassPool。

4. ClassPool.getDefault()方法会返回默认的唯一的实例

5. new ClassPool(true). 会创建一个新的对象

6. new ClassPool(parent) 可以级联

7. ClassPool.get() 方法流程, 委托给 get0(String classname, boolean useCache)

   1. ```java
      synchronized CtClass get0(String classname, boolean useCache){
          CtClass clazz = null;
          if (useCache) {
              clazz = getCached(classname);
              if (clazz != null)
                  return clazz;
          }
      
          if (!childFirstLookup && parent != null) {
              clazz = parent.get0(classname, useCache);
              if (clazz != null)
                  return clazz;
          }
      
          clazz = createCtClass(classname, useCache);
          if (clazz != null) {
              // clazz.getName() != classname if classname is "[L<name>;".
              if (useCache)
                  cacheCtClass(clazz.getName(), clazz, false);
      
              return clazz;
          }
      
          if (childFirstLookup && parent != null)
              clazz = parent.get0(classname, useCache);
      
          return clazz;
      }
      ```

      1. 先从缓存中找.   这里使用了一个同步的HashTable结构   Hashtable classes
      2. 根据当前的设定, 字段 boolean childFirstLookup(Determines the search order.)  如果是先从Parent找(!childFirstLookup), 则递归调用.  否则创建CtClass对象.
      3. 

      ```java
       CtClass createCtClass(String classname, boolean useCache) 
      
      if (classname.charAt(0) == '[')
          classname = Descriptor.toClassName(classname);
      
      if (classname.endsWith("[]")) {
          String base = classname.substring(0, classname.indexOf('['));
          if ((!useCache || getCached(base) == null) && find(base) == null)
              return null;
          else
              return new CtArray(classname, this);
      }
      else
          if (find(classname) == null)
              return null;
          else
              return new CtClassType(classname, this);
      ```

      1. 这里 会根据名称是否是 '[]' 结尾, 来决定是否生成的对象是CtArray. 
      2. 创建的时候不会判断 useCache

   2. 再去parent中寻找. 

8. CtClass 是一个抽象类. 代表了一个java class

   1. ![image-20200817113320597](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200817113320597.png)

   2.  拷贝一个已经存在的类来定义一个新的类 调用 setName(name)

      1. 对于CtClassType来说

         1. ```java
            public void setName(String name) throws RuntimeException {
                String oldname = getName();
                if (name.equals(oldname))
                    return;
            
                // check this in advance although classNameChanged() below does.
                classPool.checkNotFrozen(name);
                ClassFile cf = getClassFile2();
                super.setName(name);
                cf.setName(name);
                nameReplaced();
                classPool.classNameChanged(oldname, this);
            }
            ```

         2.  同时会改变 ClassFile和 常量池中的名称

   3. ```java
      public Class toClass(ClassLoader loader, ProtectionDomain domain)
          throws CannotCompileException
      {
          ClassPool cp = getClassPool();
          if (loader == null)
              loader = cp.getClassLoader();
      
          return cp.toClass(this, loader, domain);
      }
      ```

      1. 可以使用自定义的ClassLoader来加载新生成的class

9.  ClassFile 代表一个java的字节码文件(.class file)

10. ###  使用 javassist.Loader

    1. ```java
       Loader extends ClassLoader
       ```

       1. 继承了ClassLoader

    2. ```java
       interface Translator{
           start(ClassPool pool) 在add到ClassPool的时候被调用
           onLoad(ClassPool pool, String classname)  加载某个类的时候被调用
       }
       ```

       1. 可以添加给Loader, 用来在类加载时附加监听机制