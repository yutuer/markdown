### javaFX

1. bind机制.  

   1. 当依赖变得无效或者改变的时候, 所有的监听会被通知
   2. bind可以是延迟绑定或者是eager binding
      1. eager binding 是在依赖改变之后, 绑定的变量会立即重新计算
      2. 延迟绑定不会立即计算, 会在下次read的时候重新计算
      3. 延迟绑定性能更好
   3. 绑定可以是单向或者双向
      1. 单向: 改变依赖的值, 传播给绑定的变量
      2. 双向绑定: 依赖和绑定的变量彼此同步.  一般的, 双向绑定只会在2个变量间定义
         1. ui组件要和底层数据双向绑定

2. JavaFx中的属性绑定

   1. javaFx中的所有属性都是 observable的

   2. javaFx中, 属性可以代表一个值或者一组值

   3. javaFx中, 属性是对象

   4. IntegerProperty , DoubleProperty,StringProperty都是抽象类. 每一个都有2个具体实现: 1个代表了读写属性, 一个代表了只读属性的包装.

      比如:StringDoubleProperty 和 ReadOnlyDoubleWrapper是2个具体实现

   5. 属性类提供了2个成对的方法get()/set()表示基本类型,   getValue()/setValue()表示对象类型.

   6. 属性类有三部分信息:

      1. 引用 bean,  默认值为null
      2. 名称 name. 不提供, 默认为空字符串
      3. 值  value
      4. 每个属性类提供了 getBean()和getName()的方法. 用于获取 bean和属性名称

3. 使用JavaFx beans

   1. 所有的属性都实现了 ReadOnlyProperty 接口

4. 延迟初始化属性对象

   1. 用来优化内存

   2. 使用场景

      1. 大部分只使用默认值
      2. 不适用绑定的特性

   3. ![image-20200820172619081](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200820172619081.png)

   4. Observable接口是顶层接口. 用来观察内容变的无效.   可以注册监听

      1. 处理属性 Invalidation 事件.  在javaFx中使用延迟解析
      2. 当一个invalid属性再次变的invalid, 也不会产生Invalidation 事件
      3. 在重新计算的时候会变得valid. 例如调用了get()或者getValue()方法

   5. ObservableValue 接口继承了 Observable接口. 可以用来观察变化, 当值变化的时候, 会有一个changed event发生.   有一个getValue()的方法返回wrap的值. 可以注册监听器ChangeListener.  监听器 changed()方法接受三个参数(ObservableValue 对象, 旧值, 新值)

      1. InvalidationListener 

         1. ```java
            public void addListener(InvalidationListener listener) {
                helper = ExpressionHelper.addListener(helper, this, listener);
            }
            ```

      2. ChangeListener 

         1. ```java
            public void addListener(ChangeListener<? super Number> listener) {
                helper = ExpressionHelper.addListener(helper, this, listener);
            }
            ```

      3. 都是观察者模式.   监听器成为一个链表

         1.  当有事件发生的时候 , 会调用辅助方法来从头触发事件

         2. ```java
            {@link ExpressionHelper}
            public static <T> void fireValueChangedEvent(ExpressionHelper<T> helper) {
                if (helper != null) {
                    helper.fireValueChangedEvent();
                }
            }
            ```

   6. Property接口 添加了5个bind接口

   7. **避免内存泄漏, 需要调用removeListener** .   对于一些局部添加的的addListener, 可以使用 weak Listener, 一个weak listener是一个WeakListener接口的实例

      1. ![image-20200820213054579](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200820213054579.png)

   