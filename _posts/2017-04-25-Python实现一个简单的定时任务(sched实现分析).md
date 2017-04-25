---
category: Python
---

在项目中经常有一些定时执行某项任务的情景，如定时清理过期订单等。如果项目比较小，可以自己实现这个定时任务，不必要依靠第三方库，使用Python的标准库sched即可。

实现每天凌晨4点运行任务的例子：

```python
import time
from datetime import datetime
import sched


def perform_task():
    scheduler.enter(60*60*24, 0, perform_task)
    print('hello world')


if __name__ == '__main__':
    scheduler = sched.scheduler(time.time, time.sleep)
    now = datetime.now()
    sched_time = datetime(now.year, now.month, now.day, 4, 0, 0)
    if sched_time < now:
        sched_time = sched_time.replace(day=now.day+1)
    scheduler.enterabs(sched_time.timestamp(), 0, perform_task)  # datetime.timestamp()是python3.3后才有
    print('crontab run')
    scheduler.run()
```


`sched`库的使用非常简单，首先构造一个`sched.scheduler`类，接受两个参数：timefunc和delayfunc，timefunc应该返回一个数字，代表当前时间，delayfunc函数接受一个参数，用于暂停运行的时间单元。默认一般都是time.time和time.sleep，也可以自己实现时间暂停的函数。

`scheduler`类准备好了，即可使用它提供的函数：

- **scheduler.enterabs(time, priority, action, argument=(), kwargs={})**
    插入一项任务，`time`是绝对时间，`priorty`为优先级，越小优先级越大，在两个任务在相同的时间时，取优先级大的先运行，`action`即需要执行的函数，`argument`和`kwargs`分别是函数的位置和关键字参数。
- **scheduler.enter(delay, priority, action, argument=(), kwargs={})**
    与`enterabs`不同的是，第一个是延迟运行的秒数，其他与`enterabs`一致。
- **scheduler.cancel(event)**
    从队列中取消任务
- **scheduler.queue**
    返回队列中的任务
- **scheduler.run(blocking=True)**
    运行队列任务，`blocking`选项是3.3版后添加的

上述小脚本实现定时任务的关键是调度了`perform_task`时，使用`scheduler.enter(60*60*24, 0, perform_task)`再插入一项任务自己后才真正实行。这样队列中一直有任务，`scheduler.run()`将无限的运行到天荒地老。

标准库`sched`的实现其实很简单，一共不到100行代码。

```python
# 首先准备一个命名元组保存任务的属性
Event = namedtuple('Event', 'time, priority, action, argument')

class scheduler:
    def __init__(self, timefunc, delayfunc):
        """Initialize a new instance, passing the time and delay
        functions"""
        self._queue = []  # 这个列表即用来保存任务队列
        self.timefunc = timefunc
        self.delayfunc = delayfunc

    def enterabs(self, time, priority, action, argument):
        # enterabs根据传入的参数构建一个任务元组，使用标准库heapq堆的插入函数heappush
        # 将新的任务插入队列，headppush函数会根据time和priority参数将任务放到正确的队
        # 列中。即self._queue是一个优先级队列（堆）
        event = Event(time, priority, action, argument)
        heapq.heappush(self._queue, event)
        return event

    def enter(self, delay, priority, action, argument):
        # 将当前时间加上延迟之后即使用enterabs插入队列
        time = self.timefunc() + delay
        return self.enterabs(time, priority, action, argument)

    def run(self):
        # 这里就是sched调度任务的代码
        q = self._queue
        delayfunc = self.delayfunc
        timefunc = self.timefunc
        pop = heapq.heappop
        while q:
            time, priority, action, argument = checked_event = q[0]  # q[0]就是堆的最上层
            now = timefunc()
            if now < time:
                delayfunc(time - now)
            else:
                event = pop(q)
                # Verify that the event was not removed or altered
                # by another thread after we last looked at q[0].
                if event is checked_event:
                    action(*argument)
                    delayfunc(0)   # Let other threads run
                else:
                    heapq.heappush(q, event)
```

上面是Python2.7中的`sched`标准库，Python3中增加了锁，保证线程安全。

两者的源代码在：

python2: https://hg.python.org/cpython/file/2.7/Lib/sched.py
python3: https://hg.python.org/cpython/file/3.6/Lib/sched.py
