##### Broker

1. broker 启动

   1. 启动 Packet, Message, Web消息的注册
   2. 启动MainProcess 线程池
   3. 读取网络配置
   4. 从主服读取配置
   5. 获取 Broker 主逻辑service  MasterService, 加入到MainProcess中
   6. 启动多个WorkerService 用来处理和其他服务连接的请求处理

2. 

3.  MasterService 会有很多的http处理

4. 当有新连接进入的时候. 

5. ```java
   @Override
   public void newConnected(AbstractLoginObject object)
   {
       ServerObject serverObject = (ServerObject) object;
       int innerServerID = (int) serverObject.getUniqueID();
       serverObject.setInnerServerID(innerServerID);
       GlobalInnerServerConfig config = configMap.get(innerServerID);
       if (config == null)
       {
           serverObject.closeChannel();
           return;
       }
       serverObject.setServerType(config.getServerType());
       waitAddQueue.add(serverObject);
   }
   ```

6. 

   ```java
   class MasterService
   {
       public void tick(int interval)
       {
           tickConfig(interval);
   
           tickWaitAdd(interval);
   
           tickRegister(interval);
   
           tickFuture(interval);
   
           tickReconnect(interval);
   
           tickHttpClient(interval);
       }
   }
   ```

   

   

   ```java
   private void tickConfig(int interval)
   {
       loadConfigRemainTime -= interval;
       if (loadConfigRemainTime <= 0)
       {
           // TODO 改成异步load
           // load配置
           ConfigHttpCallback callback = new ConfigHttpCallback();
           callback.setMasterService(this);
           loadConfigRemainTime = LOAD_CONFIG_TIME;
   
           configs.clear();
           NetConfig.getInstance().loadGlobalInnerServerConfig(configs);
   
           refreshConfig(configs);
       }
   }
   ```



```java
/**
 * tick处理待注册
 */
private void tickWaitAdd(int interval)
{
    if (waitAddQueue.isEmpty())
    {
        return;
    }

    for (int i = 0; i < 32; i++)
    {
        ServerObject serverObject = waitAddQueue.poll();
        if (serverObject == null)
        {
            break;
        }

        waitRegisterList.add(serverObject);

        // 发送注册消息
        sendRegisterPacket(serverObject);
    }

}
```



```java
/**
 * tick处理网络服务对象
 * @param interval
 */
private void tickRegister(int interval)
{
    if (waitRegisterList.isEmpty())
    {
        return;
    }

    int size = waitRegisterList.size();
    for (int i = size - 1; i >= 0; i--)
    {
        ServerObject serverObject = waitRegisterList.get(i);
        if (serverObject == null)
        {
            waitRegisterList.remove(i);
            continue;
        }

        // 保证把消息先发出去
        serverObject.processOutput();

        TIntHashSet alreadySend = new TIntHashSet();
        boolean isRegistered = serverObject.processRegister();

        if (serverObject.getExceptionEnum() != ObjectExceptionEnum.NONE)
        {
            CoreLog.CORE_COMMON.warn("broker连接到内网服务:{} 在注册阶段异常:{}", serverObject.getInnerServerID(), serverObject.getExceptionEnum());
            waitRegisterList.remove(i);
            configMap.remove(serverObject.getInnerServerID());
            return;
        }

        if (isRegistered)
        {
            // 注册成功
            waitRegisterList.remove(i);
            int innerServerID = serverObject.getInnerServerID();

            serverObjectMap.put(innerServerID, serverObject);
            alreadySend.add(innerServerID);

            // 派发给worker线程
            InnerServerRegisterNotice notice = new InnerServerRegisterNotice(MessageIDConst.MW_INNER_SERVER_REGISTER_NOTICE);
            notice.setServerObject(serverObject);
            TIntIntHashMap map = notice.getAllInnerServerMap();

            TIntObjectIterator<ServerObject> iterator = serverObjectMap.iterator();
            while (iterator.hasNext())
            {
                iterator.advance();
                ServerObject object = iterator.value();
                if (alreadySend.contains(object.getInnerServerID()))
                {
                    continue;
                }
                map.put(object.getInnerServerID(), object.getServerType());
            }

            NewInnerServerNotice newNotice = new NewInnerServerNotice(MessageIDConst.B_NEW_INNSER_SERVER_NOTICE);
            newNotice.setInnerServerID(innerServerID);
            newNotice.setServerType(serverObject.getServerType());
            newNotice.setMaxOnlineCount(serverObject.getMaxOnlineCount());
            BrokerGlobals.broadcastMessageToAllWorker(this, newNotice);

            WorkerService workerService = BrokerGlobals.getInstance().getModWorkerService(innerServerID);
            sendMessage(workerService, notice);

            continue;
        }

    }

}
```





```java
/**
     * tick处理结果
     * @param interval
     */
    private void tickFuture(int interval)
    {
        if (futureMap.isEmpty())
        {
            return;
        }

        TIntObjectIterator<ChannelFuture> iterator = futureMap.iterator();
        while (iterator.hasNext())
        {
            iterator.advance();

            ChannelFuture channelFuture = iterator.value();
            if (channelFuture.isDone())
            {
                if (channelFuture.isSuccess())
                {
                    CoreLog.CORE_COMMON.info("netty client连接成功, innerServer:{}", iterator.key());
                    iterator.remove();
                    continue;
                }
                else
                {
                    // netty的channel connect失败了
//                    CoreLog.CORE_COMMON.warn("netty client连接失败,等待重试 innerServer:{}", iterator.key());
                    waitReconnectArray.add(iterator.key());
                    iterator.remove();
                    continue;
                }
            }
        }
        if (futureMap.isEmpty())
        {
            CoreLog.CORE_COMMON.warn("netty client连接失败,等待重试 innerServerMap:{}", futureMap.keys());
        }
    }
```



```java
/**
 * 处理重连
 * @param interval
 */
private void tickReconnect(int interval)
{
    if (waitReconnectArray.isEmpty())
    {
        return;
    }

    reconnectRemainTime -= interval;
    if (reconnectRemainTime <= 0)
    {
        reconnectRemainTime = RECONNECT_TIME;
    }
    else
    {
        return;
    }

    int size = waitReconnectArray.size();
    for (int i = size - 1; i >= 0; i--)
    {
        int innerServerID = waitReconnectArray.get(i);
        GlobalInnerServerConfig config = configMap.get(innerServerID);
        if (config == null)
        {
            continue;
        }
        addNewInnerServer(config);
        waitReconnectArray.remove(i);
    }
}
```



```java
private void tickHttpClient(int interval)
{
    for (int i = 0; i < 10; i++)
    {
        BrokerHttpCallback httpCallback = httpClientModule.pop();
        if (httpCallback == null)
        {
            break;
        }

        try
        {
            httpCallback.callback();
        }
        catch (Exception e)
        {
            CoreLog.CORE_COMMON.warn("tick处理http client异常, e:{}", ExceptionUtils.exceptionToString(e));
        }
    }
}
```