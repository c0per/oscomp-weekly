---
marp: true
---

# 4.9 OS 比赛交流

## ArceOS 代码阅读

陈嘉钰

---

# 进展

- 阅读 `ArceOS` 应用的代码，写介绍文档。

- `ArceOS` 中对 `task` 的管理、同步相关的代码 `sync`。

---

# ArceOS 初赛

- 没有特权级概念，全部代码均运行在S级。

- 没有`syscall`概念，用户库（`libax`）直接调用`ArceOS module`。

- 没有进程概念，仅存在线程（`task`）。

- 没有加载进程的过程。

---

# 接下来的目标

- 继续了解`ArceOS`的代码

- 思考如何添加多特权级支持

下周：尝试添加U特权级

- 修改`axruntime`，在调用用户`main`前切换至U态。

- 修改`libax`，添加一个简单的`syscall`（`Instant::now()`）。

- 暂时不考虑进程，S和U共用一个页表。

---

# `RunQueue`

`RunQueue` 包含一个 `Scheduler`，保存全部的`TaskInner`。

`IDLE_TASK`：一个 idle 进程，与 `rCore` 中的类似。

```rust
fn block_current(); // 由用户提供 WaitQueue，自行负责唤醒
fn sleep_until(); // 使用内核中的timers，定时唤醒

fn unblock_task();
fn exit_current();
fn add_task();
```

初始化时，生成`IDLE_TASK`和一个名为`main`的`init task`。

其他的`task`通过调用`libax::spawn`生成。

---

# Scheduler

- 合作式的`FifoScheduler`

- 抢占式的`RRScheduler`

```rust
fn add_task(&mut self, task: Self::SchedItem);
fn remove_task(&mut self, task: &Self::SchedItem) -> Option<Self::SchedItem>;
fn pick_next_task(&mut self) -> Option<Self::SchedItem>;
fn put_prev_task(&mut self, prev: Self::SchedItem, preempt: bool);
fn task_tick(&mut self, current: &Self::SchedItem) -> bool;
```

---

# `task` 调度

## `resched_inner(preempt: bool)`

- 将当前`task`设为`Ready`状态，并放回`Scheduler`。

- 调用`pick_next_task(preempt)`取出下一个`task`并`switch`。

---

# `preempt` 抢占

`TaskInner` 中添加的属性：

- `need_resched`：需要重新调度。

- `preempt_disable_count`：禁用`preemt`次数。

`timer` 中断：

- `scheduler_timer_tick()`，记录当前`task`的运行时间（`time slice`）。

- RR：时间片执行完毕后，标记`need_resched`。

`unblock task`：

- 标记`need_resched`。

---

# `preempt` 抢占

## `resched()`

- 如果启用了`preempt`并且`need_resched`，调用`resched_inner(true)`。

- 否则标记`need_resched`。

在每次调用`enable_preempt()`时，若彻底启动了抢占，则调用`resched()`。

---

# `WaitQueue`

被`block`的`task`被放在一个用户态的`WaitQueue`中等待唤醒。

`WaitQueue::notify()` 可以唤醒其中的 `task`。

内核中存在全局的`WaitQueue`：`TIMER_LIST`。
`timers::check_events()`，检查`TIMER_LIST`中是否有需要唤醒的`task`。

---

# `sync` 中的 `spinlock`

- `raw`

- 关 `preempt`

- 关中断和 `preempt`

---

# 添加特权级

- `axhal`：启动引导相关代码，无需修改

- `axruntime`：修改`rust_main`，进入U特权级。

- 添加`trap handler`，在S态中执行调度、`syscall`。

- 暂时不支持`task::spawn`。

- 修改`libax`，调用`syscall`。
