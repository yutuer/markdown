### 邮件系统review

1. Scene  

   1. **场景的类名字不是Actor开头的..**

       ```java
      class SceneMailModule   
      ```

   2.  字段

      ```java
      /**
           * 私人邮件列表（按接收时间从前到后排序）
           */
          private ArrayList<ActorMail> actorMails = new ArrayList<>();
      
          /**
           * 当前邮件索引值
           */
          private long currentMailIndex = 0;
      
          /**
           * 当前用户已经领取过的全服邮件索引
           */
          private long currentServerMailIndex = 0;
      
          /**
           * 第一个过期邮件时间(秒)，避免每次tick遍历查找过期邮件
           */
          private int firstExpireMailTimeSec = 0;
      ```



​			  

```java
		class AbstractMail
```

​					

```java
/**
     * 本邮件对应的邮件配置,运营邮件的这个字段是从
     * 运营工具或者db中取到的,其他情况是走的配置
     */
    private DictMailConfig mailConfig;

    /**
     * 携带的元宝个数
     */
    private int gold;

    /**
     * 携带的银票个数
     */
    private int receipt;

    /**
     * 携带的银两个数
     */
    private int silver;

    /**
     * 道具数据
     */
    private ArrayList<BPMailItem> items = new ArrayList<>(BPCommonConst.MAX_MAIL_ITEM);

    /**
     * 邮件的索引id
     */
    private long mailIndexID;

    /**
     * 邮件内容,如果是配置的邮件,里面存放的是(x,y,z…………)格式,每个占位符
     * 之间是有逗号分隔，如果是运营邮件,content就是真实的邮件内容
     */
    private String content;

    /**
     * 邮件标题,如果是配置的邮件,里面存放的是(x,y,z…………)格式,每个占位符
     * 之间是有逗号分隔，如果是运营邮件,title就是真实的邮件标题
     */
    private String title;

    /**
     * 邮件的发送时间
     */
    private int sendTime;
```



1. World

   1. 类名    

      ```java
      class WorldMailModule
      ```

   2. 字段 

      ```java
      /**
       * 运营发送的全服邮件列表
       */
      private ArrayList<ServerMail> serverMailArray = new ArrayList<>();
      
      /**
       * 待添加的全服邮件
       */
      private ArrayList<ServerMail> addServerMailArray = new ArrayList<>();
      
      /**
       * 待删除的全服邮件
       */
      private TLongHashSet deleteServerMailArray = new TLongHashSet();
      
      /**
       * 当前的全服邮件最新索引
       * (为什么要用long? serverMailArray的size最大也是int)
       * (能否这个值使用检查过后的索引的含义? 否则会有可能每次都需要去重新检查条件过滤)
       */
      private long currentServerMailIndex = 0;
      
      /**
       * 玩家id -> 邮件列表
       */
      private TLongObjectHashMap<LinkedList<ActorOfflineMail>> actorId2OfflineMailListMap = new TLongObjectHashMap<>();
      
      /**
       * 邮件id -> 离线邮件
       */
      private TLongObjectHashMap<ActorOfflineMail> offlineMailMap = new TLongObjectHashMap<>();
      
      /**
       * 待删除的离线邮件集合,保存了邮件的mailIndexID
       */
      private TLongHashSet deleteOfflineMailIndexIDs = new TLongHashSet();
      
      /**
       * 待添加的离线邮件集合,保存了邮件数据
       */
      private LinkedList<ActorOfflineMail> addOfflineMailList = new LinkedList<>();
      
      /**
       * 当前的玩家离线邮件最新索引
       */
      private long currentOfflineMailIndex = 0;
      
      /**
       * 全服邮件 邮件的索引id对应list下标
       */
      private TLongIntHashMap serverMailIndexMap = new TLongIntHashMap();
      ```

2. packet:

   1. 获取邮件列表. 

   2. 读取一封邮件

      1.  根据mailIndex找到一封邮件 (现在是遍历所有, 能否优化)
      2.  如果读取过, 直接返回, 不报错
      3.  设置读取过, 设置读取邮件时间
      4.  发送客户端读取结果

   3. 领取附件.  **注意和读取邮件重合的逻辑**

      1.  注意此方法和 一键领取附件有重合的地方, 为了共用,  **有的地方的逻辑并不一定正确**
      2.  没有附件就返回. (这里没有设置过领取状态, 所以可以一直领取. 客户端应该会做发送协议限制吧)
      3.  判断能否领取放进背包(邮件的只能往背包放), 
         1. 有错误返回.
      4.  领取到背包, 加物品, 加钱.
      5.  **如果邮件没有读取, 则读取邮件** (这个其实是单个领取附件的协议 , 和一键领取有冲突的地方, 此逻辑可能错误)
      6.  设置已经领取附件状态. 保存
      7.  发送邮件(参数控制是否告诉客户端协议, 一键的话不会发送, 单独领取会发送)
      8.  根据邮件配置, 领取成功后决定是否删除邮件
      9.  如果是好友发送的邮件**(某个指定的邮件id)**, 则领取后发送好友频道聊天(**但是双方此时不一定是好友了**, 频道发送会不会报错呢)

   4. 一键领取附件.

      1. **这里的领取邮件索引s 由客户端传入**

      2. ```java
         int idSize = mailIndexIds.size();
         // 邮件使用倒序获取, 客户端参数正序 
         ArrayList<ActorMail> list = new ArrayList<>(size);
         for (int i = size - 1, j = 0; i >= 0 && j < idSize; i--)
         {
             ActorMail mail = actorMails.get(i);
             if (mail.getMailIndexID() == mailIndexIds.get(j))
             {
                 j++;
         
                 // 如果邮件有附件, 则加入
                 if (mail.isHasAttachment())
                 {
                     list.add(mail);
                 }
             }
         }
         ```

         看这段循环的逻辑.  服务器用倒序循环, 客户端传入的索引用正序, 那么需要**客户端传入的参数也是倒序的**

      3. 使用上面的逻辑, 挑出有附件的邮件. 然后使用倒序删除(其实没必要), 这里的邮件列表是拷贝的.. 可能顾忌到领取单个附件会有删除邮件的情况吧

      4. 发送给客户端一键领取结果协议

   5. 删除邮件

      1. 判断是否可以删除
         1. 必须已读并且没有附件或者附件已经领取成功才可以删除, 否则不能删除
      2. 删除邮件逻辑

   6. 删除所有邮件

      1. **倒序遍历** ,  删除会从列表中移除. 

      

      SceneMailModule:

      1. addMail(): 发送邮件给玩家

         1. 设置邮件receiveTime
         2. 保存
         3. 计算最近过期的邮件时间
         4. 推送红点
         5. 如果满了, 删除最旧的并且未带附件的邮件

      2.  copyFrom方法:

         1.  数据库中得到玩家已有的顺序的mailList 列表
         2.  找出当前邮件最大的 currentMailIndex值
         3.  得到最近过期的邮件时间

         

      

      World:

      ​	1. 全局邮件.  

         		保存在  serverMailArray中,  支持add, delete操作.    为了支持delete操作, 需要一个额外的值currentServerMailIndex来充当当前索引

         		为了支持快速查找索引, 所以增加了serverMailIndexMap的数据结构. 用来serverMailIndex记录的对应索引.  但是因为支持了delete操作,所以其实这里有bug.  说明如下:

      1.    删除某个全局邮件之后, 这里的索引也需要删除. 并且因为serverMailArray中少了一个, 所以需要更新这个索引之后的全部索引
        2. 如果刚刚好, 删除的正好是某个玩家上次查看到的索引. 这里的索引删除后, 当下次玩家来查找对应的时候, 这里的值相当于没有设置.

           所以解决方法就是, 去掉这个map, 每个玩家全局遍历. 找到   [玩家传到world的上次读取的值, 当前currentServerMailIndex) 中间的这些邮件发回给玩家, 同时更新玩家的全局邮件的currentServerMailIndex

      !! 对于刚创建的玩家 需要判断下邮件发送时间, 只返回创建时间之后的服务器所发的邮件

      2. 离线邮件
         1. 如果玩家不在线, 则将邮件放入到 actorId2OfflineMailListMap(方便Actor获取)和offlineMailMap(方便根据索引查找, 目前功能中没啥用)中
         2. 记得存库
         3. 服务器加载后, 循环所有离线邮件, 更新currentOfflineMailIndex索引
         4. 玩家上线后. 从actorId2OfflineMailListMap中得到离线邮件, 发送给场景

   ​	

   ​     