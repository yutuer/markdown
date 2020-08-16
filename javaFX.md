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

   1. 

      