---
layout:     post
title:      Chromium Threading and Task
date:       2020-01-10
author:     Hope
header-img: img/google-chrome-logo.jpg
catalog: true
tags:
    - Chromium
    - Task
    - Thread
---

# Chromium Threading and Task

**本文代码基于80.0.3987分支**<br>
在Chromium中，不仅有大名鼎鼎的[多进程架构](https://www.chromium.org/developers/design-documents/multi-process-architecture)，还有自己分工明确的多线程架构。在Chromium的多线程架构中，是以Task作为调度的基本单位。

[TOC]

## Chromium 多线程设计思想

- The main goal is to keep the main thread (a.k.a. “UI” thread in the browser process) and IO thread (each process' thread for handling IPC) responsive. This means offloading any blocking I/O or other expensive operations to other threads.
（主要目的是为了让主线程(例如 Browser 进程中的 UI 线程)和 IO 线程（进程中处理 IPC 消息的线程）保持快速响应。这意味着要把任何I/O阻塞和其他昂贵的操作，放在其他线程里面执行。可以放在是 devicated thread 或者 threadpool ）
- Locks are also an option to synchronize access but in Chrome we strongly prefer sequences to locks.（锁是一种实现同步访问的选择，但是在 Chrome 里面，我们更喜欢使用 sequences，而不是锁）
- Most threads have a loop that gets tasks from a queue and runs them (the queue may be shared between multiple threads).（大量的线程有一个可以从队列里面获取任务并执行它们的 loop ）

## 线程的类型

在一个进程中，往往有如下几种线程：

- 一个 main thread
    - 在 Browser 进程中 (BrowserThread::UI)：用于更新 UI
    - 在 Render 进程中：运行Blink
- 一个 io thread
    - 在 Browser 进程中(BrowserThread::IO): 用于处理 IPC 消息以及网络请求
    - 在 Render 进程中：用于处理IPC消息
- 一些使用 base::Tread 创建的，有特殊用途的线程（可能存在）
- 一些在使用线程池时产生的线程（可能存在）

**注意，在 Chromium，我们应该是用 base::Thread 来创建线程,而不是每个平台下的特定的 platform thread**

## Task

task 封装了一个 `base::OnceCloseure`。我们会通过 `TaskRunner` 把 task 放进一个 `TaskQueue` 里面，以至于能一直执行相对应的方法。

### Task 的执行方式

- Parallel：Task 会以并行、无序地执行，可能同一时间内在任意线程上执行所有任务（通过线程池实现）
- Sequenced：Task 会以加入的顺序执行，同一时间内在任意线程上执行一个任务（通过 `SequencedTaskRunner` 实现）
- Single Threaded：Task 会以加入的顺序执行，同一时间内只在固定的线程上执行一个任务(通过 `SingleSuqenceTaskRunner` 实现)

#### Posting a Task

- Posting to the Thread Pool：
通过 [base/task/post_task.h](https://source.chromium.org/chromium/chromium/src/+/master:base/task/thread_pool.h)文件中定义的 `base::PostTask()` 来实现（在84版本代码中，还可以通过 `base::ThreadPool::PostTask()` 实现。
    ```
    base::PostTask(FROM_HERE, base::BindOnce(&Task));
    base::PostTask(
        FROM_HERE, {base::TaskPriority::BEST_EFFORT, MayBlock()},
        base::BindOnce(&Task));
    ```
- Posting to the TaskRunner:
    ```
    auto* task_runner = base::CreateTaskRunner({base::TaskPriority::USER_VISIBLE});
    task_runner->PostTask(FROM_HERE, base::BindOnce(&Task));
    ```
    定义在 通过定义在[base/task/post_task.h](https://source.chromium.org/chromium/chromium/src/+/master:base/task/thread_pool.h) 文件 `base::CreateTaskRunner()` 来获取 `TaskRunner`。
- Posting a Sequenced Task
  ```
  auto* task_runner = base::CreateSequencedTaskRunner({base::TaskPriority::USER_VISIBLE});
  task_runner->PostTask(FROM_HERE, base::BindOnce(&Task));
  ```
- Posting to the Current (Virtual) Thread
  ```
  base::SequencedTaskRunnerHandle::Get()->PostTask(
    FROM_HERE, base::BindOnce(&Task);
  ```
- Posting to the single Thread
  ```
  auto* task_runner = base::CreateSingleThreadTaskRunner({base::TaskPriority::USER_VISIBLE});
  task_runner->PostTask(FROM_HERE, base::BindOnce(&Task));
  ```

**注意，在使用 `base::BindOnce()` 方法产生一个 `base::OnceClosure` 的时候，一般会传递一个 `base::WeakPrt`,而不是一个裸指针。`base::WeakPrt`能确保所指向的对象销毁时，绑定在对象上的回调能被取消。否则一般会产生一个段错误**

## 底层原理

从上述的代码可以看出，我们可以通过各种 `TaskRunner` 的子类来 post task。那么task 是怎么被调度，运行的呢？下图是具体的类图。<br>
[![image](https://raw.githubusercontent.com/YuWeiCong/draw.io/master/chromium/images/Task%20uml.jpg)](https://raw.githubusercontent.com/YuWeiCong/draw.io/master/chromium/images/Task%20uml.jpg)

### MessageLoop

整个消息循环的入口，会初始化 `MessagePump`、`SequenceManager`、`TaskQueue` 等对象,并调用其`BindToCurrentThread`方法。一般 `base::Thread` 都会有一个 `MessageLoopCurrent` 实例。但因为因为主线程并不是由 `base::Thread` 类型，所以必须手动声明一个 `MessageLoop`。

### MessagePump

顾名思义，`MessagePump`是消息泵，循环地处理与其所关联的消息（承担着消息循环的功能）。经典的消息循环如下：
```
for (;;) {
  bool did_work = DoInternalWork();
  if (should_quit_)
    break;

  did_work |= delegate_->DoWork();
  if (should_quit_)
    break;

  TimeTicks next_time;
  did_work |= delegate_->DoDelayedWork(&next_time);
  if (should_quit_)
    break;

  if (did_work)
    continue;

  did_work = delegate_->DoIdleWork();
  if (should_quit_)
    break;

  if (did_work)
    continue;

  WaitForWork();
}
```

### ThreadController

获取要被调度的 task，并传递给TaskAnnotator。
`ThreadController::SechuleWork()` 方法会从 `SequencdTaskSource::SelectNextTask`中获取下一个要被调动的 task，并调用 `TaskAnnotator::RunTask()`方法执行该 task

### SequencedTaskSource

为 `ThreadControll` 提供要被调度的 task，这个 task 来自于 `TaskQueueSelector::SelectWorkQueueToService()`

### TaskQueueSelector

`TaskQueueSelector` 会从当前的 `TaskQueue` 中的 `immediate_work_queue` 和 `delayed_incoming_queue` 中获取相对应的 task

### TaskQueue

Task 保存在 `TaskQueue` 里面，  `TaskQueue`的具体实现里面，有四个队列：
- Immediate (non-delayed) tasks:
    - immediate_incoming_queue - PostTask enqueues tasks here.
    - immediate_work_queue - SequenceManager takes immediate tasks here.
- Delayed tasks
    - delayed_incoming_queue - PostDelayedTask enqueues tasks here.
    - delayed_work_queue - SequenceManager takes delayed tasks here.

当 `immediate_work_queue` 为空的时候，`immediate_work_queue` 会和 `immediate_incoming_queue` 进行交换。delayed task 都会先保存在 `delayed_incoming_queue`，然后会有个名为`TimeDomain`类，会把到了时间的 delayed task 加入到 `delayed_work_queue`中。

上述过程中的流程图表示如下：
[![image](https://raw.githubusercontent.com/YuWeiCong/draw.io/master/chromium/images/Task%20flow.jpg)](https://raw.githubusercontent.com/YuWeiCong/draw.io/master/chromium/images/Task%20flow.jpg)


## 参考文档
- [Chromium线程模型、消息循环](https://blog.csdn.net/wy5761/article/details/44095089)
- [Threading and Tasks in Chrome](https://source.chromium.org/chromium/chromium/src/+/master:docs/threading_and_tasks.md;l=1?q=threading_and_tasks.md&sq=&ss=chromium)