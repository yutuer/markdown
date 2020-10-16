###  SceneFlowModule

```java
class SceneFlowModule
{
    /**
     * 当前所有的触发器
     */
    private ArrayDeque<AbstractSceneTrigger> triggers = new ArrayDeque<>();

    /**
     * 待加入的场景触发器
     */
    private ArrayList<AbstractSceneTrigger> waitAddTriggers = new ArrayList<>();

    /**
     * 待删除的互斥触发器id
     */
    private TIntHashSet waitRemoveMutuallyTriggerIDs = new TIntHashSet();

    /**
     * 监听器map
     * key: 触发器类型, value:监听器
     */
    private TIntObjectHashMap<LinkedList<SceneListener>> listenerMap = new TIntObjectHashMap<>();

    /**
     * 已执行的行为Id集合,action全推加入该集合（断线重连需要重新推送）
     */
    private TIntArrayList executeActionIdList = new TIntArrayList();

    /**
     * 已执行的行为Id集合,action只推最后一个的，加入该集合（断线重连需要重新推送）
     * key为行为类型，value为对应的map集合
     */
    private TIntObjectHashMap<TIntIntHashMap> lastExecuteActionInfoMap = new TIntObjectHashMap<>();

    /**
     * 玩家对应等待结束的actionId集合，actorId--->actionId---->行为是否执行状态(0:未执行，1：执行)
     * 例如：每个玩家播放剧情动画行为，需要等动画结束后触发事件，如果断线没有执行，上线需要重新推送
     */
    private TLongObjectHashMap<TIntIntHashMap> actorIdToActionWaitFinishDataMap = new TLongObjectHashMap<>();

    /**
     * 场景带有参数消息action加入该map缓存, key为actionId, value为BPStruct.PBActionIdData
     */
    private TIntObjectHashMap<BPStruct.PBActionIdData.Builder> actionInfoDataMap = new TIntObjectHashMap<>();

    /**
     * 当前trigger显示进度到目标追踪面板缓存结构
     */
    private BPStruct.PBTriggerShowPanelData.Builder triggerPanelDataBuilder = BPStruct.PBTriggerShowPanelData.newBuilder();

    /**
     * trigger显示进度到星级条缓存map，key为triggerId, value为BPStruct.PBTriggerShowPanelData结构数据
     */
    private TIntObjectHashMap<BPStruct.PBTriggerShowPanelData.Builder> triggerStarDataMap = new TIntObjectHashMap<>();

    /**
     * 场景循环器数组
     */
    private ArrayList<AbstractSceneLooper> sceneLooperList = new ArrayList<>();

    /**
     * 场景选择器数组
     */
    private ArrayList<AbstractSceneSelector> sceneSelectorList = new ArrayList<>();

    /**
     * 场景条件数组
     */
    private ArrayList<AbstractSceneCondition> sceneConditionList = new ArrayList<>();

    /**
     * 场景多路触发器组
     */
    private ArrayList<AbstractSceneMultichannel> sceneMultichannelList = new ArrayList<>();

    /**
     * 场景交互器数组 (数据量上去需要改成TIntHashMap)
     */
    private ArrayList<AbstractSceneInteractor> sceneInteractorList = new ArrayList<>();

    /**
     * 删除的looper(用作缓存的)
     */
    private ArrayList<AbstractSceneLooper> deleteLoopers = new ArrayList<>();

    /**
     * 删除的selector(用作缓存的)
     */
    private ArrayList<AbstractSceneSelector> deleteSelecters = new ArrayList<>();

    /**
     * 删除的interactor（用作缓存）
     */
    private ArrayList<AbstractSceneInteractor> deleteInteractors = new ArrayList<>();

    /**
     * 全局triggerId集合
     */
    private TIntHashSet globalTriggerIdSet = new TIntHashSet();    
}

```



```java
public void tickTrigger(int interval)
{
    if (triggers.isEmpty())
    {
        return;
    }

    int size = triggers.size();
    for (int i = 0; i < size; i++)
    {
        AbstractSceneTrigger trigger = triggers.pollFirst();
        if (trigger == null)
        {
            continue;
        }

        int maxTriggerCount = trigger.getMaxTriggerCount();
        // 是否已经达到了最大触发次数, 达到了就不需要维护触发器了
        int beforeCount = trigger.getTriggerCount();
        if (beforeCount >= maxTriggerCount)
        {
            // 销毁触发器
            trigger.destroyTrigger();
            continue;
        }

        DictSceneTriggerData dictSceneTriggerData = trigger.getDictSceneTriggerData();
        if (!waitRemoveMutuallyTriggerIDs.isEmpty())
        {
            if (waitRemoveMutuallyTriggerIDs.contains(dictSceneTriggerData.getTriggerId()))
            {
                trigger.destroyTrigger();
                continue;
            }
        }

        trigger.addTickInterval(interval);
        int tickInterval = trigger.getTickInterval();
        if (tickInterval >= trigger.getFrameRate())
        {
            //特别注意一个tick里可能执行多次触发
            trigger.tick(interval);
            trigger.setTickInterval(0);
        }

        // 是否已经达到了最大触发次数, 达到了就不需要维护触发器了
        int afterCount = trigger.getTriggerCount();

        // 本次触发了
        if (afterCount - beforeCount > 0)
        {
            // condition的触发, 在这里trigger
            int conditionSize = sceneConditionList.size();
            for (int j = 0; j < conditionSize; j++)
            {
                AbstractSceneCondition condition = sceneConditionList.get(j);
                if (condition != null)
                {
                    condition.trigger(trigger.getTriggerID());
                }
            }
        }

        if (afterCount < maxTriggerCount)
        {
            triggers.offerLast(trigger);
        }
        else
        {
            // 销毁触发器
            trigger.destroyTrigger();
        }
    }
}
```



```java
class AbstractSceneTrigger
{
    /**
     * 触发器类型
     */
    protected SceneTriggerTypeEnum triggerType = SceneTriggerTypeEnum.NONE;

    /**
     * 触发器参数
     */
    protected int[] params = null;

    /**
     * 触发的行为id数组
     */
    protected int[] actionArray = null;

    /**
     * 所在场景
     */
    protected AbstractBPScene scene = null;

    /**
     * 触发次数
     */
    private int triggerCount = 0;

    /**
     * trigger 的配置
     */
    protected DictSceneTriggerData dictSceneTriggerData = null;

    /**
     * 关联的选择器
     */
    private AbstractSceneSelector selector = null;

    /**
     * 累计的tick时间间隔
     */
    private int tickInterval = 0;

    /**
     * 是否显示进度到目标面板（0:不显示,  1:追踪面板,  2:星级面板,  3：追踪和星级, 4:骑马小游戏）
     */
    protected int isShowPanel;    
}

```











```java
class AbstractSceneSelector
```

```java
/**
 * 配置引用
 */
private DictSceneSelector dictSceneSelector;

private boolean destroy = false;

/**
 * 所在场景
 */
private AbstractBPScene scene;

/**
 * 触发次数
 */
private int triggerCount;
```



```java
class AbstractSceneLooper
```

```java
/**
 * 配置引用
 */
private DictSceneLooper dictSceneLooper;

/**
 * 当前循环执行次数
 */
private int currentLoopCount = 0;

/**
 * 所在场景
 */
private AbstractBPScene scene;

/**
 * 是否已销毁
 */
private boolean destroy = false;
```





```java

```

```java
class AbstractSceneCondition
{
     /**
     * 配置引用
     */
    private DictSceneCondition dictSceneCondition;

    /**
     * 关联的循环器
     */
    private AbstractSceneLooper looper = null;

    /**
     * 未完成的trigger集合1
     * value: trigger id
     */
    private TIntHashSet unfinishedTriggerSet1 = new TIntHashSet();

    /**
     * 未完成的action集合1
     * value: action id
     */
    private TIntHashSet unfinishedActionSet1 = new TIntHashSet();

    /**
     * 未完成的trigger集合2
     * value: trigger id
     */
    private TIntHashSet unfinishedTriggerSet2 = new TIntHashSet();

    /**
     * 未完成的action集合2
     * value: action id
     */
    private TIntHashSet unfinishedActionSet2 = new TIntHashSet();

    /**
     * 所关心的行为集合
     */
    private TIntHashSet careActionSet = new TIntHashSet();

    /**
     * 所关心的触发器集合
     */
    private TIntHashSet careTriggerSet = new TIntHashSet();

    /**
     * 所在场景
     */
    private AbstractBPScene scene;

    /**
     * 是否需要销毁
     */
    private boolean destroy = false;

    // 
    void trigger(int triggerID);

    //
    void action(int actionID);

    //
    void conditionDone(); 
}
```



```java
class AbstractSceneAction
{
    /**
     * 行为类型
     */
    protected SceneActionTypeEnum actionType = SceneActionTypeEnum.NONE;

    /**
     * 参数
     */
    private int[] params = null;

    /**
     * 所在场景
     */
    protected AbstractBPScene scene = null;

    // 参数的传递. 因为Action可能各种各样, 所以这里其实传递对象最有优势. 
    // 如果传递不同的参数. 对于Action[] 的形式, 怎么传递参数?
    abstract public int execute(int triggerObjectID, int param1, int param2, int param3, int param4); 
    
    public int getParam(int index);
}
```

