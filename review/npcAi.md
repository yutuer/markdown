### NpcAiModule

通用AI管理器

```java
class NpcIdleAIManager
```

```java
private Npc npc;

/**
 * 表格数据
 * 后面的优先级更高
 */
private IdleAiState[] aiStates;

/**
 * 条件标志是否更新
 */
private boolean isDirty;

/**
 * 各个状态的条件状态
 */
private boolean[][] conditions;

/**
 * key事件类型，value 关联的条件索引 (状态索引，条件索引) 一组
 */
private EnumMap<AIEventEnum,TIntArrayList> eventToCondition = new EnumMap<>(AIEventEnum.class);

/**
 * 当前运行的状态索引
 */
private int curStateIndex;

/**
 * 条件触发器
 */
private List<AbstractAiConditionTrigger> triggers = new ArrayList<>();

/**
 * 条件触发器
 * key: 触发器类型, value:条件触发器
 */
private EnumMap<AiConditionTriggerEnum, List<AbstractAiConditionTrigger>> triggerMap = new EnumMap(AiConditionTriggerEnum.class);

// 添加触发器
void addTrigger(int stateIndex, int conditionIndex, DictNpcIdleAiConditionData data){}

void tick(int interval);

// 检查所有条件
void checkAiCondition();

```



```java
 abstract class AbstractAiConditionTrigger
```

```java
protected NpcIdleAIManager idleAIManager;

/**
 * 状态索引
 */
protected int stateIndex;

/**
 * 条件索引
 */
protected int conditionIndex;

/**
 * 参数
 */
protected DictNpcIdleAiConditionData conditionData;


void init(NpcIdleAIManager idleAIManager, int stateIndex, int conditionIndex, DictNpcIdleAiConditionData data);
```



```java
class IdleAiState<T extends AbstractCharacter>
```

```java
private T owner;

/**
 * 所属状态索引
 */
private int stateIndex;

/**
 * 空闲ai
 */
private NpcIdleAIManager idleAIManager;

/**
 * 状态数据
 */
private DictNpcIdleAiStateData stateData;

/**
 * 进入条件
 */
private AbstractAiCondition[] conditions;

/**
 * 执行行为
 */
private AbstractAiAction[] actions;

/**
 * 状态切换，执行行为 (已经执行过， 不再执行)
 */
private AbstractAiAction[] changeActions;

/**
 * 状态切换时,退出执行的行为id （一定执行）
 */
private AbstractAiAction[] exitActions;

/**
 * 当前运行行为索引
 */
private int curActionIndex = -1;

/**
 * 当前状态是否被打断
 */
private boolean isBreakCurState;

//高优先级状态达成，打断当前状态
//1、执行打断行为
//2、执行切换状态行为
//3、执行退出状态行为
void breakAiState();
```





```java
 abstract class AbstractAiAction<T extends AbstractCharacter>
```

```java
protected T owner;

/**
 * 行为配置数据
 */
protected DictNpcIdleAiActionData actionData;

/**
 * 是否执行
 * 默认执行，若执行过，则不在执行
 */
protected boolean isRun;

/**
 * 当前行为是否完成
 */
protected boolean isComplete;

/**
 * 打断行为
 */
protected AbstractAiAction breakAction;
```





