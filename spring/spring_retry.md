|            | **模块**                    | **验证**                                 |
| ---------- | --------------------------- | ---------------------------------------- |
| **Step 1** | `RetryContext` (接口与实现) | 状态变量（count, lastException）的存储。 |
| **Step 2** | `RetryPolicy` (简单的实现)  | `canRetry` 方法的布尔逻辑判断。          |
| **Step 3** | `BackOffPolicy`             | `Thread.sleep` 的时长计算是否正确。      |
| **Step 4** | **`RetryTemplate`**         | 验证 `while` 循环和 `try-catch` 逻辑。   |
| **Step 5** |                             |                                          |

### 1 核心

**`RetryPolicy` (重试策略)**：决定是否应该重试。它知道当前重试了多少次，发生了什么异常。

**`BackOffPolicy` (退避策略)**：决定何时重试。例如，立刻重试，还是等待 1 秒，还是指数级增加等待时间（1s, 2s, 4s...）？

**`RetryContext` (重试上下文)**：贯穿整个重试过程的数据包，记录了重试次数、最后一次异常等状态。

### 2 执行引擎 : RetryTemplate

`src/support/RetryTemplate.java`




---

#### 1 RetryPolicy (重试策略) - 接口类

`RetryOperations.excute()` 开始一次重试序列，调用 `open(parent)` 得到 `RetryContext`。

每次尝试执行业务逻辑 `RetryCallBack` 前，调用 `canRetry(context)` ，查看 `context` 中的内容决定是否继续。

业务回调执行失败时，调用 `registerThrowable(context, throwable)` 更新状态；框架可能再调用 `canRetry`。

重试序列完成或放弃时，调用 `close(context)` 做清理。

##### RetryContext

继承AttributeAccessor，可以在重试的各阶段(`RetryListener`、`BackOffPolicy`)之间传递自定义数据。

定义了常量( NAME, STATE_KEY 等)，储存元**数据**，记录了一次重试操作过程中的所有实时**状态**。

由 `RetryPolicy.open()` 创建，并在整个重试循环中被传递，最后由 `RetryPolicy.close()` 结束。

##### RetryOpration

定义如何**执行**带有重试的业务逻辑。负责调度各个重试的策略。
```java
//仅传入retryCallback，失败抛出异常
<T, E extends Throwable> T execute(RetryCallback<T, E> retryCallback) throws E;

//带恢复的执行，重试次数耗尽后执行回复逻辑
<T, E extends Throwable> T execute(RetryCallback<T, E> retryCallback, RecoveryCallback<T> recoveryCallback) throws E;

//有状态的重试，
<T, E extends Throwable> T execute(RetryCallback<T, E> retryCallback, RetryState retryState) throws E, ExhaustedRetryException;

//有状态有恢复的重试
<T, E extends Throwable> T execute(RetryCallback<T, E> retryCallback, RecoveryCallback<T> recoveryCallback, RetryState retryState) throws E;
```



##### 嵌套重试



#### 2 BackOffPolicy (退避策略)



#### 3 RetryContext (重试上下文)
