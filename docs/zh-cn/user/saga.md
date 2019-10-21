# SEATA Saga 模式
## 概述
Saga模式是SEATA提供的长事务解决方案，在Saga模式中，业务流程中每个参与者都提交本地事务，当出现某一个参与者失败则补偿前面已经成功的参与者，一阶段正向服务和二阶段补偿服务都由业务开发实现。

![Saga模式示意图](https://github.com/long187/seata.github.io/blob/develop/img/saga/sagas.png?raw=true)

理论基础：Hector & Kenneth 发表论⽂ Sagas （1987）

### 适用场景：
* 业务流程长、业务流程多
* 参与者包含其它公司或遗留系统服务，无法提供 TCC 模式要求的三个接口

### 优势：
* 一阶段提交本地事务，无锁，高性能
* 事件驱动架构，参与者可异步执行，高吞吐
* 补偿服务易于实现

### 缺点：
* 不保证隔离性（应对方案见后面文档）

### Saga的实现：
#### 基于状态机引擎的 Saga 实现：

目前SEATA提供的Saga模式是基于状态机引擎来实现的，机制是：
  1. 通过状态图来定义服务调用的流程并生成 json 状态语言定义文件
  2. 状态图中一个节点可以是调用一个服务，节点可以配置它的补偿节点
  3. 状态图 json 由状态机引擎驱动执行，当出现异常时状态引擎反向执行已成功节点对应的补偿节点将事务回滚
   > 注意: 异常发生时是否进行补偿也可由用户自定义决定
  4. 可以实现服务编排需求，支持单项选择、并发、子流程、参数转换、参数映射、服务执行状态判断、异常捕获等功能

![示例状态图](https://github.com/long187/seata.github.io/blob/develop/img/saga/demo_statelang.png?raw=true)

## 快速开始

### Demo简介
基于dubbo构建的微服务下，使用Saga模式演示分布式事务的提交和回滚；
业务流程图如下图所示：
![demo业务流程图](https://github.com/long187/seata.github.io/blob/develop/img/saga/demo_business_process.png?raw=true)

先下载seata-samples工程：https://github.com/seata/seata-samples.git

> 注意SEATA版本需要0.9.0以上

在dubbo-saga-sample中一个分布式事务内会有2个Saga事务参与者，分别是: [InventoryAction](https://github.com/seata/seata-samples/blob/master/saga/dubbo-saga-sample/src/main/java/io/seata/samples/saga/action/InventoryAction.java) 和 [BalanceAction](https://github.com/seata/seata-samples/blob/master/saga/dubbo-saga-sample/src/main/java/io/seata/samples/saga/action/BalanceAction.java) ;分布式事务提交则两者均提交，分布式事务回滚则两者均回滚；

这2个Saga参与者均是 dubbo 服务，两个参与都有一个reduce方法，表示库存扣减或余额扣减，还有一个compensateReduce方法，表示补偿扣减操作。

- InventoryAction 接口定义如下：

```java
public interface InventoryAction {

    /**
     * reduce
     * @param businessKey
     * @param amount
     * @param params
     * @return
     */
    boolean reduce(String businessKey, BigDecimal amount, Map<String, Object> params);

    /**
     * compensateReduce
     * @param businessKey
     * @param params
     * @return
     */
    boolean compensateReduce(String businessKey, Map<String, Object> params);
}
```

- 这个场景用状态语言定义就是下面的json：src/main/resources/statelang/reduce_inventory_and_balance.json
```json
{
    "Name": "reduceInventoryAndBalance",
    "Comment": "reduce inventory then reduce balance in a transaction",
    "StartState": "ReduceInventory",
    "Version": "0.0.1",
    "States": {
        "ReduceInventory": {
            "Type": "ServiceTask",
            "ServiceName": "inventoryAction",
            "ServiceMethod": "reduce",
            "CompensateState": "CompensateReduceInventory",
            "Next": "ChoiceState",
            "Input": [
                "$.[businessKey]",
                "$.[count]"
            ],
            "Output": {
                "reduceInventoryResult": "$.#root"
            },
            "Status": {
                "#root == true": "SU",
                "#root == false": "FA",
                "$Exception{java.lang.Throwable}": "UN"
            }
        },
        "ChoiceState":{
            "Type": "Choice",
            "Choices":[
                {
                    "Expression":"[reduceInventoryResult] == true",
                    "Next":"ReduceBalance"
                }
            ],
            "Default":"Fail"
        },
        "ReduceBalance": {
            "Type": "ServiceTask",
            "ServiceName": "balanceAction",
            "ServiceMethod": "reduce",
            "CompensateState": "CompensateReduceBalance",
            "Input": [
                "$.[businessKey]",
                "$.[amount]",
                {
                    "throwException" : "$.[mockReduceBalanceFail]"
                }
            ],
            "Output": {
                "compensateReduceBalanceResult": "$.#root"
            },
            "Status": {
                "#root == true": "SU",
                "#root == false": "FA",
                "$Exception{java.lang.Throwable}": "UN"
            },
            "Catch": [
                {
                    "Exceptions": [
                        "java.lang.Throwable"
                    ],
                    "Next": "CompensationTrigger"
                }
            ],
            "Next": "Succeed"
        },
        "CompensateReduceInventory": {
            "Type": "ServiceTask",
            "ServiceName": "inventoryAction",
            "ServiceMethod": "compensateReduce",
            "Input": [
                "$.[businessKey]"
            ]
        },
        "CompensateReduceBalance": {
            "Type": "ServiceTask",
            "ServiceName": "balanceAction",
            "ServiceMethod": "compensateReduce",
            "Input": [
                "$.[businessKey]"
            ]
        },
        "CompensationTrigger": {
            "Type": "CompensationTrigger",
            "Next": "Fail"
        },
        "Succeed": {
            "Type":"Succeed"
        },
        "Fail": {
            "Type":"Fail",
            "ErrorCode": "PURCHASE_FAILED",
            "Message": "purchase failed"
        }
    }
}
```

该json表示的状态图:
![该json表示的状态图](https://github.com/long187/seata.github.io/blob/develop/img/saga/demo_statelang.png?raw=true)

状态语言在一定程度上参考了[AWS Step Functions](https://docs.aws.amazon.com/zh_cn/step-functions/latest/dg/tutorial-creating-lambda-state-machine.html)

#### "状态机" 属性简介:
* Name: 表示状态机的名称，必须唯一
* Comment: 状态机的描述
* Version: 状态机定义版本
* StartState: 启动时运行的第一个"状态"
* States: 状态列表，是一个map结构，key是"状态"的名称，在状态机内必须唯一

#### "状态" 属性简介:
* Type: "状态" 的类型，比如有:
  * ServiceTask: 执行调用服务任务
  * Choice: 单条件选择路由
  * CompensationTrigger: 触发补偿流程
  * Succeed: 状态机正常结束
  * Fail: 状态机异常结束
  * SubStateMachine: 调用子状态机
  * CompensateSubMachine: 用于补偿一个子状态机
* ServiceName: 服务名称，通常是服务的beanId
* ServiceMethod: 服务方法名称
* CompensateState: 该"状态"的补偿"状态"
* Input: 调用服务的输入参数列表, 是一个数组, 对应于服务方法的参数列表, $.表示使用表达式从状态机上下文中取参数，表达使用的[SpringEL](https://docs.spring.io/spring/docs/4.3.10.RELEASE/spring-framework-reference/html/expressions.html), 如果是常量直接写值即可
* Ouput: 将服务返回的参数赋值到状态机上下文中, 是一个map结构，key为放入到状态机上文时的key（状态机上下文也是一个map），value中$.是表示SpringEL表达式，表示从服务的返回参数中取值，#root表示服务的整个返回参数
* Status: 服务执行状态映射，框架定义了三个状态，SU 成功、FA 失败、UN 未知, 我们需要把服务执行的状态映射成这三个状态，帮助框架判断整个事务的一致性，是一个map结构，key是条件表达式，一般是取服务的返回值或抛出的异常进行判断，默认是SpringEL表达式判断服务返回参数，带$Exception{开头表示判断异常类型。，value是当这个条件表达式成立时则将服务执行状态映射成这个值
* Catch: 捕获到异常后的路由
* Next: 服务执行完成后下一个执行的"状态"
* Choices: Choice类型的"状态"里, 可选的分支列表, 分支中的Expression为SpringEL表达式, Next为当表达式成立时执行的下一个"状态"
* ErrorCode: Fail类型"状态"的错误码
* Message: Fail类型"状态"的错误信息

更多详细的状态语言解释请看[State language referance](#State-language-referance)章节


### demo 运行指南

#### step 1 启动 SEATA Server

运行 [SeataServerStarter](https://github.com/seata/seata-samples/blob/master/saga/sofarpc-saga-sample/src/test/java/io/seata/samples/saga/SeataServerStarter.java) ，启动 Seata Server；

#### step 2 启动 dubbo provider Demo

运行 [DubboSagaProviderStarter](https://github.com/seata/seata-samples/blob/master/saga/dubbo-saga-sample/src/test/java/io/seata/samples/saga/starter/DubboSagaProviderStarter.java) ，启动 dubbo provider；

#### step 3 启动 Saga Demo

运行 [DubboSagaTransactionStarter](https://github.com/seata/seata-samples/blob/master/saga/dubbo-saga-sample/src/main/java/io/seata/samples/saga/starter/DubboSagaTransactionStarter.java) , 启动 demo工程；


与这个Demo一起的还有调用本地服务和调用SOFA RPC服务的示例

## 设计
### 状态机引擎原理:

![状态机引擎原理](https://github.com/long187/seata.github.io/blob/develop/img/saga/saga_engine_mechanism.png?raw=true)

* 图中的状态图是先执行stateA, 再执行stataB，然后执行stateC
* "状态"的执行是基于事件驱动的模型，stataA执行完成后，会产生路由消息放入EventQueue，事件消费端从EventQueue取出消息，执行stateB
* 在整个状态机启动时会调用Seata Server开启分布式事务，并生产xid, 然后记录"状态机实例"启动事件到本地数据库
* 当执行到一个"状态"时会调用Seata Server注册分支事务，并生产branchId, 然后记录"状态实例"开始执行事件到本地数据库
* 当一个"状态"执行完成后会记录"状态实例"执行结束事件到本地数据库, 然后调用Seata Server上报分支事务的状态
* 当整个状态机执行完成, 会记录"状态机实例"执行完成事件到本地数据库, 然后调用Seata Server提交或回滚分布式事务

### 状态机引擎设计:

![状态机引擎设计](https://github.com/long187/seata.github.io/blob/develop/img/saga/saga_engine.png?raw=true)

状态机引擎的设计主要分成三层, 上层依赖下层，从下往上分别是：
* Eventing 层:
  * 实现事件驱动架构, 可以压入事件, 并由消费端消费事件, 本层不关心事件是什么消费端执行什么，由上层实现
* ProcessController 层:
  * 由于上层的Eventing驱动一个“空”流程执行的执行，"state"的行为和路由都未实现, 由上层实现
* StateMachineEngine 层:
  * 实现状态机引擎每种state的行为和路由逻辑
  * 提供 API、状态机语言仓库


## 最佳实践

### Saga 服务设计的实践经验
#### 允许空补偿
* 空补偿：原服务未执行，补偿服务执行了
* 出现原因：
  * 原服务 超时（丢包）
  * Saga 事务触发 回滚
  * 未收到 原服务请求，先收到 补偿请求

所以服务设计时需要允许空补偿, 即没有找到要补偿的业务主键时返回补偿成功并将原业务主键记录下来

#### 防悬挂控制
* 悬挂：补偿服务 比 原服务 先执行
* 出现原因：
  * 原服务 超时（拥堵）
  * Saga 事务回滚，触发 回滚
  * 拥堵的 原服务 到达

所以要检查当前业务主键是否已经在空补偿记录下来的业务主键中存在，如果存在则要拒绝服务的执行

### 缺乏隔离性的应对
* 由于 Saga 事务不保证隔离性, 在极端情况下可能由于脏写无法完成回滚操作, 比如举一个极端的例子, 分布式事务内先给用户A充值, 然后给用户B扣减余额, 如果在给A用户充值成功, 在事务提交以前, A用户把线消费掉了, 如果事务发生回滚, 这时则没有办法进行补偿了。这就是缺乏隔离性造成的典型的问题, 实践中一般的应对方法是：
  * 业务流程设计时遵循“宁可长款, 不可短款”的原则, 长款意思是客户少了线机构多了钱, 以机构信誉可以给客户退款, 反之则是短款, 少的线可能追不回来了。所以在业务流程设计上一定是先扣款。
  * 有些业务场景可以允许让业务最终成功, 在回滚不了的情况下可以继续重试完成后面的流程, 所以状态机引擎除了提供“回滚”能力还需要提供“向前”恢复上下文继续执行的能力, 让业务最终执行成功, 达到最终一致性的目的。


#### 幂等控制
* 原服务与补偿服务都需要保证幂等性, 由于网络可能超时, 可以设置重试策略，重试发生时要通过幂等控制避免业务数据重复更新

## API referance

#### StateMachineEngine
``` java
public interface StateMachineEngine {

    /**
     * start a state machine instance
     * @param stateMachineName
     * @param tenantId
     * @param startParams
     * @return
     * @throws EngineExecutionException
     */
    StateMachineInstance start(String stateMachineName, String tenantId, Map<String, Object> startParams) throws EngineExecutionException;

    /**
     * start a state machine instance with businessKey
     * @param stateMachineName
     * @param tenantId
     * @param businessKey
     * @param startParams
     * @return
     * @throws EngineExecutionException
     */
    StateMachineInstance startWithBusinessKey(String stateMachineName, String tenantId, String businessKey, Map<String, Object> startParams) throws EngineExecutionException;

    /**
     * start a state machine instance asynchronously
     * @param stateMachineName
     * @param tenantId
     * @param startParams
     * @param callback
     * @return
     * @throws EngineExecutionException
     */
    StateMachineInstance startAsync(String stateMachineName, String tenantId, Map<String, Object> startParams, AsyncCallback callback) throws EngineExecutionException;

    /**
     * start a state machine instance asynchronously with businessKey
     * @param stateMachineName
     * @param tenantId
     * @param businessKey
     * @param startParams
     * @param callback
     * @return
     * @throws EngineExecutionException
     */
    StateMachineInstance startWithBusinessKeyAsync(String stateMachineName, String tenantId, String businessKey, Map<String, Object> startParams, AsyncCallback callback) throws EngineExecutionException;

    /**
     * forward restart a failed state machine instance
     * @param stateMachineInstId
     * @param replaceParams
     * @return
     * @throws ForwardInvalidException
     */
    StateMachineInstance forward(String stateMachineInstId, Map<String, Object> replaceParams) throws ForwardInvalidException;

    /**
     * forward restart a failed state machine instance asynchronously
     * @param stateMachineInstId
     * @param replaceParams
     * @param callback
     * @return
     * @throws ForwardInvalidException
     */
    StateMachineInstance forwardAsync(String stateMachineInstId, Map<String, Object> replaceParams, AsyncCallback callback) throws ForwardInvalidException;

    /**
     * compensate a state machine instance
     * @param stateMachineInstId
     * @param replaceParams
     * @return
     * @throws EngineExecutionException
     */
    StateMachineInstance compensate(String stateMachineInstId, Map<String, Object> replaceParams) throws EngineExecutionException;

    /**
     * compensate a state machine instance asynchronously
     * @param stateMachineInstId
     * @param replaceParams
     * @param callback
     * @return
     * @throws EngineExecutionException
     */
    StateMachineInstance compensateAsync(String stateMachineInstId, Map<String, Object> replaceParams, AsyncCallback callback) throws EngineExecutionException;

    /**
     * skip current failed state instance and forward restart state machine instance
     * @param stateMachineInstId
     * @return
     * @throws EngineExecutionException
     */
    StateMachineInstance skipAndForward(String stateMachineInstId) throws EngineExecutionException;

    /**
     * skip current failed state instance and forward restart state machine instance asynchronously
     * @param stateMachineInstId
     * @param callback
     * @return
     * @throws EngineExecutionException
     */
    StateMachineInstance skipAndForwardAsync(String stateMachineInstId, AsyncCallback callback) throws EngineExecutionException;

    /**
     * get state machine configurations
     * @return
     */
    StateMachineConfig getStateMachineConfig();
}
```

## Config referance
#### config a StateMachineEngine in spring config xml
``` xml
<bean id="stateMachineEngine" class="io.seata.saga.engine.impl.ProcessCtrlStateMachineEngine">
        <property name="stateMachineConfig" ref="dbStateMachineConfig"></property>
</bean>
<bean id="dbStateMachineConfig" class="io.seata.saga.engine.config.DbStateMachineConfig">
    <property name="dataSource" ref="dataSource"></property>
    <property name="resources" value="statelang/*.json"></property>
    <property name="enableAsync" value="true"></property>
    <property name="threadPoolExecutor" ref="threadExecutor"></property><!-- 事件驱动执行时使用的线程池, 如果所有状态机都同步执行可以不需要 -->
    <property name="applicationId" value="saga_sample"></property>
    <property name="txServiceGroup" value="my_test_tx_group"></property>
</bean>
<bean id="threadExecutor"
        class="org.springframework.scheduling.concurrent.ThreadPoolExecutorFactoryBean">
    <property name="threadNamePrefix" value="SAGA_ASYNC_EXE_" />
    <property name="corePoolSize" value="1" />
    <property name="maxPoolSize" value="20" />
</bean>

<!-- Seata Server进行事务恢复时需要通过这个Holder拿到stateMachineEngine实例 -->
<bean class="io.seata.saga.rm.StateMachineEngineHolder">
    <property name="stateMachineEngine" ref="stateMachineEngine"/>
</bean>
```

## State language referance
### "状态机"的属性列表
```json
{
    "Name": "reduceInventoryAndBalance",
    "Comment": "reduce inventory then reduce balance in a transaction",
    "StartState": "ReduceInventory",
    "Version": "0.0.1",
    "States": {
    }
}
```
* Name: 表示状态机的名称，必须唯一
* Comment: 状态机的描述
* Version: 状态机定义版本
* StartState: 启动时运行的第一个"状态"
* States: 状态列表，是一个map结构，key是"状态"的名称，在状态机内必须唯一, value是一个map结构表示"状态"的属性列表

### 各种"状态"的属性列表
#### ServiceTask: 
```json
"States": {
    ...
    "ReduceBalance": {
    "Type": "ServiceTask",
    "ServiceName": "balanceAction",
    "ServiceMethod": "reduce",
    "CompensateState": "CompensateReduceBalance",
    "Input": [
        "$.[businessKey]",
        "$.[amount]",
        {
            "throwException" : "$.[mockReduceBalanceFail]"
        }
    ],
    "Output": {
        "compensateReduceBalanceResult": "$.#root"
    },
    "Status": {
        "#root == true": "SU",
        "#root == false": "FA",
        "$Exception{java.lang.Throwable}": "UN"
    },
    "Catch": [
        {
            "Exceptions": [
                "java.lang.Throwable"
            ],
            "Next": "CompensationTrigger"
        }
    ],
    "Next": "Succeed"
}
    ...
}
```
* ServiceName: 服务名称，通常是服务的beanId
* ServiceMethod: 服务方法名称
* CompensateState: 该"状态"的补偿"状态"
* IsForUpdate: 标识该服务会更新数据, 默认是false, 如果配置了CompensateState则默认是true, 有补偿服务的服务肯定是数据更新类服务
* IsPersist: 执行日志是否进行存储, 默认是true, 有一些查询类的服务可以配置在true, 执行日志不进行存储提高性能, 因为当异常恢复时可以重复执行
* Input: 调用服务的输入参数列表, 是一个数组, 对应于服务方法的参数列表, $.表示使用表达式从状态机上下文中取参数，表达使用的[SpringEL](https://docs.spring.io/spring/docs/4.3.10.RELEASE/spring-framework-reference/html/expressions.html), 如果是常量直接写值即可。复杂的参数如何传入见:
* Ouput: 将服务返回的参数赋值到状态机上下文中, 是一个map结构，key为放入到状态机上文时的key（状态机上下文也是一个map），value中$.是表示SpringEL表达式，表示从服务的返回参数中取值，#root表示服务的整个返回参数
* Status: 服务执行状态映射，框架定义了三个状态，SU 成功、FA 失败、UN 未知, 我们需要把服务执行的状态映射成这三个状态，帮助框架判断整个事务的一致性，是一个map结构，key是条件表达式，一般是取服务的返回值或抛出的异常进行判断，默认是SpringEL表达式判断服务返回参数，带$Exception{开头表示判断异常类型。，value是当这个条件表达式成立时则将服务执行状态映射成这个值
* Catch: 捕获到异常后的路由
* Next: 服务执行完成后下一个执行的"状态"

> 当没有配置Status对服务执行状态进行映射, 系统会自动判断状态: 
> * 没有异常则认为执行成功, 
> * 如果有异常, 则判断异常是不是网路连接超时, 如果是则认为是FA
> * 如果是其它异常, 服务IsForUpdate=true则状态为UN, 否则为FA

> 整个状态机的执行状态如果判断？是由框架自己判断的：
> * 如果所有服务执行成功（事务提交成功）则为成功
> * 如果补偿成功（事务回滚成功）则为FA
> * 如果有更新类服务未成功且没有进行补偿（事务提交失败） 则为UN
> * 发生补偿且有未补偿成功的服务（回滚失败）则为UN
> * 存在事务提交或回滚失败的情况Seata Sever都会不断发起重试

#### Choice: 
```java
"ChoiceState":{
    "Type": "Choice",
    "Choices":[
        {
            "Expression":"[reduceInventoryResult] == true",
            "Next":"ReduceBalance"
        }
    ],
    "Default":"Fail"
}
```
Choice类型的"状态"是单项选择路由
Choices: 可选的分支列表, 只会选择第一个条件成立的分支
Expression: SpringEL表达式
Next: 当Expression表达式成立时执行的下一个"状态"

#### CompensationTrigger:
```java
"CompensationTrigger": {
    "Type": "CompensationTrigger",
    "Next": "Fail"
}
```
CompensationTrigger类型的state是用于触发补偿事件, 回滚分布式事务
Next: 补偿成功后路由到的state


#### SubStateMachine:
```json
"CallSubStateMachine": {
    "Type": "SubStateMachine",
    "StateMachineName": "simpleCompensationStateMachine",
    "CompensateState": "CompensateSubMachine",
    "Input": [
        {
            "a": "$.1",
            "barThrowException": "$.[barThrowException]",
            "fooThrowException": "$.[fooThrowException]",
            "compensateFooThrowException": "$.[compensateFooThrowException]"
        }
    ],
    "Output": {
        "fooResult": "$.#root"
    },
    "Next": "Succeed"
}
```
SubStateMachine类型的"状态"是调用子状态机
StateMachineName: 要调用的子状态机名称
CompensateState: 子状态机的补偿state, 可以不配置, 系统会自动创建它的补偿state, 子状态机的补偿实际就是调用子状态机的compensate方法, 所以用户并不需要自己实现一个对子状态机的补偿服务。当配置这个属性时, 可以里利用Input属性自定义传入一些变量, 见下面的CompensateSubMachine


#### CompensateSubMachine:
```json
"CompensateSubMachine": {
    "Type": "CompensateSubMachine",
    "Input": [
        {
            "compensateFooThrowException": "$.[compensateFooThrowException]"
        }
    ]
}
```
CompensateSubMachine类型的state是专门用于补偿一个子状态机的state，它会调用子状态机的compensate方法，可以利用Input属性传入一些自定义的变量, Status属性自定判断补偿是否成功

#### 复杂参数的Input定义
```json
"FirstState": {
    "Type": "ServiceTask",
    "ServiceName": "demoService",
    "ServiceMethod": "complexParameterMethod",
    "Next": "ChoiceState",
    "ParameterTypes" : ["java.lang.String", "int", "io.seata.saga.engine.mock.DemoService$People", "[Lio.seata.saga.engine.mock.DemoService$People;", "java.util.List", "java.util.Map"],
    "Input": [
        "$.[people].name",
        "$.[people].age",
        {
            "name": "$.[people].name",
            "age": "$.[people].age",
            "childrenArray": [
                {
                    "name": "$.[people].name",
                    "age": "$.[people].age"
                },
                {
                    "name": "$.[people].name",
                    "age": "$.[people].age"
                }
            ],
            "childrenList": [
                {
                    "name": "$.[people].name",
                    "age": "$.[people].age"
                },
                {
                    "name": "$.[people].name",
                    "age": "$.[people].age"
                }
            ],
            "childrenMap": {
                "lilei": {
                    "name": "$.[people].name",
                    "age": "$.[people].age"
                }
            }
        },
        [
            {
                "name": "$.[people].name",
                "age": "$.[people].age"
            },
            {
                "name": "$.[people].name",
                "age": "$.[people].age"
            }
        ],
        [
            {
                "@type": "io.seata.saga.engine.mock.DemoService$People",
                "name": "$.[people].name",
                "age": "$.[people].age"
            }
        ],
        {
            "lilei": {
                "@type": "io.seata.saga.engine.mock.DemoService$People",
                "name": "$.[people].name",
                "age": "$.[people].age"
            }
        }
    ],
    "Output": {
        "complexParameterMethodResult": "$.#root"
    }
}
```
上面的complexParameterMethod方法定义如下:
```java
People complexParameterMethod(String name, int age, People people, People[] peopleArrya, List<People> peopleList, Map<String, People> peopleMap)

class People {

    private String name;
    private int    age;

    private People[] childrenArray;
    private List<People> childrenList;
    private Map<String, People> childrenMap;

    ...
}
```
启动状态机时传入参数:
```java
Map<String, Object> paramMap = new HashMap<>(1);
People people = new People();
people.setName("lilei");
people.setAge(18);
paramMap.put("people", people);
String stateMachineName = "simpleStateMachineWithComplexParams";
StateMachineInstance inst = stateMachineEngine.start(stateMachineName, null, paramMap);
```

> 注意ParameterTypes属性是可以不用传的，调用的方法的参数列表中有Map, List这种可以带泛型的集合类型, 因为java编译会丢失泛型, 所以需要用这个属性, 同时在Input的json中对应的对这个json加"@type"来申明泛型