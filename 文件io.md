### 文件io

![img](https://pic2.zhimg.com/80/v2-deaab2ba9c2592aec7d92e87874fdcdd_720w.jpg)

1. VFS 虚拟文件系统,树结构, 内存中.  不同的节点映射到不同的物理位置(ext4, ext3  fat 甚至是网络节点)

   1.  ll
   2. df  显示挂载情况   sda3  表示sda硬盘的3分区 
      1. linux 一般看不到分区, 由目录代表了. windows可以
      2. 记录可以覆盖 
   3. mount 和 unmount
   4. 目录树结构趋向于稳定, 有一个映射的过程

2. 文件类型

   1.  冯诺依曼    cpu(运算器, 控制器)   内存(存储器)  io(输入输出设备)  
   2. VFS 一切皆文件
      1. -: 代表普通文件 (可执行, 图片, 文本)
      2. d: 代表目录
      3. l: 链接文件
         1. 软链接  
            1. ln -s a b 
            2. inode 号不一样 , 并且links 没有增加
            3. 删除原来的a后, b会报错
         2. 硬链接  磁盘里就这么一个文件, 只不过在有了不同的path 指向了同一个物理位置, 修改其中一个, 可以看到另外一个也改了
            1. 很像2个引用 引用了同一个对象
            2. ln a b  之后 查看links, 可以看到数字2, 表示硬链接引用的次数
            3. **stat** a 查看文件元数据信息  发现 ab的inode是一样的
      4. b: 块设备  来回自由漂移的读, 类似于字节数组
      5. c: 字符设备.  不能读到之前的值, 不能随便漂移
      6. s: socket
      7. p: 管道
      8. eventPoll:  一块内存区域

3. iNode 

4. PageCache 内存中  页缓存(默认4k)

   1. 2个程序打开同一个文件, pageCache共享
   2. Dirty 修改内存中数据, 修改后会设置为dirty.  会有个flush过程
      1. flush的时间, 决定了io模式
         1. 程序调用内核直接刷
         2. 交由内核决定什么时候刷 .  阈值满足后刷新(时间, 或者容量)

5. FD 文件描述符, 指针  属性: seek(偏移)    **lsof命令**

   1. 每个程序读取文件的时候, 都有自己的seek. 不修改同一个位置的时候, 不用加锁  

6. ###### 演示

   1. dd if=/dev/zero(无限大的空, 但是不占用磁盘空间) of=mydisk.img  bs=1048576(1M) count=100 得到一个被0填充的文件
   2. losetup /dev/loop0 mydisk.img  把mydisk挂到设备下面 
   3. mke2fs /dev/loop0  格式化 为ext2文件系统
   4. mount -t ext2 /dev/loop0 /mnt/ooxx  挂载
   5. 从/bin/bash 拷贝一个bash到自定义目录 /mnt/ooxx/bin下
   6. ldd bash 分析程序动态链接库(启动的时候的依赖)有哪些.
   7. 拷贝所有的依赖库  花括号扩展, 一次拷贝多个  cp /lib64{libtinfo.so.5}  ./lib64/
   8. chroot ./  根目录切换到当前目录. 并启动bash

7. lsof 显示进程打开了哪些文件  ls -p $$(表示当前bash进程)

   1. 任何程序都有0 (标准输入) 1(标准输出)  2(报错输出) 三个文件描述符
      1. /proc 内核映射路径
      2. cd /proc/$$ 进入当前进程映射目录
   2. **exec** 8<ooxx.txt  
      1. 虽然exec和source都是在父进程中直接执行，但exec这个与source有很大的区别，source是执行shell脚本，而且执行后会返回以前的shell。而exec的执行不会返回以前的shell了，而是直接把以前登陆shell作为一个程序看待，在其上经行复制
      2. ![image-20200822100427158](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200822100427158.png)
   3. **read** a 0<& 8   读取8文件描述符到a变量  read读取会读取到换行符停止
   4. **pcstat** -pid $$ 查看缓存页 pageCache
   5. **lsof** **-op** $$(或者pid) (使用前需要yum install lsof -y安装)  查看当前fd细节
      1. 在终端下输入lsof即可显示系统打开的文件，因为 lsof 需要访问核心内存和各种文件，所以必须以 root 用户的身份运行它才能够充分地发挥其功能。  
   
8. bash 解释执行程序 $$ 优先级比 | (管道符) 高, $ 要比|低.  

   1. 管道会将 两边的语句先**启动子进程** , 例如 { echo $BASHPID; read x; } | { cat; echo $BASHPID; read y; } 会先启动2个子进程
   2. echo $$ | cat 会打印当前进程id  
   3. echo $BASHPID | cat 会打印子进程id
   4. 