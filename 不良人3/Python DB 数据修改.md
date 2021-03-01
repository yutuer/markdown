### Python DB 数据修改 

1. 修改db中某个表的非二进制字段 
   1. 直接sql语句修改
2.  修改db中某个表的二进制字段 .  
   1. 方法参数为(serverId, tableName, actorId, 二进制列名称)  serverId, tableName, actorId, protobuf对象
      1. 查询出二进制字段数据, 
      2. 转化成对象A.  
      3. 转化传入的json数据为对象B
      4. 根据传入的字段索引参数  找到对应的字段.  将相应字段(比如.  A.b.c.d) 设置为B,  即  A.b.c.d = B
      5. 将A转化为二进制数据, 存入数据库
3. 组件
   1. db 组件.  
      1. 查询主服. 得到相关serverId的dataSource
      2. 根据dataSource 执行sql
   2. protoBuf文件
   3. 对象操作
      1. json转对象
      2. protobuf转对象
      3. 对象转protobuf