### Redis 源码 笔记

1.SDS

```c
/*
 * 保存字符串对象的结构
 */
struct sdshdr {
    
    // buf 中已占用空间的长度
    int len;

    // buf 中剩余可用空间的长度
    int free;

    // 数据空间
    char buf[];
};
```

1.  空间预先分配  类似jvm的TLAB的内存分配
2. 
3.  常数复杂度获取字符串长度