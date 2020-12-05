### jvm笔记

1. G1痛点:  RemSet 过大.   refine线程会占用应用cpu, 多5%-10%
2. object copy time
3. Object o = o1的时候,  在young区里面是没有些writeBarrier的
4. update remset time. 跨区引用越多, gc的时间越大
5. Cms的时间 重点看 init remark 的2个标记时间
6. gc
   1.  喜欢小的, 生命周期短的对象, 
   2. 避免大对象的分配
   3. 避免复杂的内部对象关联
   4. 避免对象池(依赖自己应用的回收)
7. JIT 发现一个父类只有一个子类的时候, 会将子类 inline. 如果这个时候又load了一个新的子类, 会造成jit的退优化
8. JIT退优化原因
   1. unstable if 不稳定的if判断. 如果jit优化了if, 但是突然else可以使用了, 会造成退优化
   2. null check. JIT发现对象不可能为null. 会优化掉
   3. class check
   4. bimorphic
9. JIT 编译级别 c1 c2等 一般是0->3->4
10. 第一次热更新, codecache会变大
11. tlab增大, 会导致YGC变快