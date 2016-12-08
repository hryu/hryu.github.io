---
layout: post
title: How a Process Sleeps and Wakes up
---

## Contents

* [Introduction][section-introduction]
* [Process States][section-process-states]
* [How Process Sleeps][section-process-sleeps]
* [How Process Wakes up][section-process-wakes-up]
* [Sleep States and Signals][section-states-signals]

---

## Introduction [section-introduction]
어떻게 프로세스가 잠들고 깨게 되는지 알아본다.

>NOTE  
>본 문서는 CFS 에 의존적이다.
>프로세스 생성/파괴 및 ptrace, 기타 스케쥴러 관련 내용은 다루지 않는다.

---

## Process States [section-process-states]

	include/linux/sched.h:
	
	...
	#define TASK_RUNNING			0
	#define TASK_INTERRUPTIBLE		1
	#define TASK_UNINTERRUPTIBLE	2
	...
	#define TASK_WAKEKILL			128
	...
	#define TASK_KILLABLE			(TASK_WAKEKILL |TASK_UNINTERRUPTIBLE)
	...
	#define TASK_NORMAL				(TASK_INTERRUPTIBLE | TASK_UNINTERRUPTIBLE)
	...

* TASK_RUNNING  
프로세스가 동작중이며 잠들지 않은 상태이다.

* TASK_INTERRUPTIBLE  
프로세스가 잠드는 상태임을 나타내며 시그널에 의해 수면이 **중단될 수 있는**
상태를 나타낸다.

* TASK_UNINTERRUPTIBLE  
프로세스가 잠드는 상태임을 나타내며 시그널에 의해 수면이 **중단될 수 없는**
상태를 나타낸다.

* TASK_WAKEKILL  
다른 상태와 or 되어 쓰이며 `TASK_UNINTERRUPTIBLE` 과 함께 쓰여 `TASK_KILLABLE`
이 된다.  

* TASK_KILLABLE  
`TASK_WAKEKILL` + `TASK_UNINTERRUPTIBLE`  
이 상태는 일반적인 시그널에 의해 중단되지는 않지만
**`SIGKILL` 에 의해 중단될 수 있는** 상태를 나타낸다.  

---

## How Process Sleeps [section-process-sleeps]

### The *Current* Process State [section-current-state]
프로세스가 잠들기 전에 먼저 실행되고 있었거나 그에 준하는 상태가 되어있어야
한다. 먼저 어떻게 어떤 프로세스가 cpu를 점유하며 실행되고 있는 상태인 `current`
가 되는지 그 상태는 어떤 것인지 알아본다.

#### Process Wakes up and Becomes TASK_RUNNING [section-process-wakes-up]
프로세스가 wake up 되고 running 상태가 된다. 두가지 경우 - 새로운 프로세스 혹은
기존 프로세스 - 가 존재한다.

##### New Process Wakes up  
fork에 의하여 새로운 프로세스가 생성된 경우

	do_fork()
		copy_process()
			sched_fork()
				__sched_fork()
					p->on_rq = 0;
					p->se.on_rq = 0;
				p->state = TASK_RUNNING;
		wake_up_new_task()
			activate_task()
				enqueue_entity()
					if (se != cfs_rq->curr)
						__enqueue_entity(cfs_rq, se);
					se->on_rq = 1;
			p->on_rq = 1;

##### Existing Process Wakes up
기존의 프로세스가 잠든 상태에 놓여 있다가 다시 깨어나는 경우

	set_current_state(TASK_INTERRUPTIBLE or UNINTERRUPTIBLE);
	if (do_i_need_to_sleep())
		schedule();
~~

	try_to_wake_up()
		ttwu_activate()
			activate_task()
				enqueue_entity()
					if (se != cfs_rq->curr)
						__enqueue_entity(cfs_rq, se);
					se->on_rq = 1;
			p->on_rq = 1;
		ttwu_do_wakeup()
			p->state = TASK_RUNNING;

`current` 프로세스가 `current`가 되기 전에는 wake up 하려는 타겟 태스크였으며 wake up 되는 태스크는 cfs_rq->curr 가 아니므로 rbtree에 들어가되고 그러므로써 다음에 호출되는 `schedule()` 에서 해당 프로세스가 실행될 기회를 가질 수 있다. 결과를 정리하면 다음과 같다.

|                        | Initial (wake up new or existing one) |
|------------------------|---------------------------------------|
| current                | -                                     |
| rq->curr               | -                                     |
| cfs_rq->curr           | -                                     |
| cfs_rq->tasks_timeline | O                                     |
| task_struct->on_rq     | O                                     |
| sched_entity->on_rq    | O                                     |
| task_struct->state     | TASK_RUNNING                          |

#### *TASK_RUNNING* Process Becomes the *Current* Process Finally...
이전에 큐에 추가된 프로세스는 `schedule()` 이 호출되었을 때 선택되어 실행될 기회를 가지게 된다.

	schedule()
		pick_next_task_fair();
			set_next_entity()
				if (se->on_rq)
					__dequeue_entity(cfs_rq, se);
				cfs_rq->curr = se;
		if (likely(prev != next))
			rq->curr = next;
			context_switch(rq, prev, next);

이전에 wake up 되어 rbtree 에 추가된 프로세스가 기회를 잡아 `pick_next_task()`에서 선택되면 `__dequeue_entity()`가 호출되어 rbtree에서 제거된다. `contex_switch()` 가 호출되고 나면 비로소 해당 프로세스는 `current`가 된다. `current` 태스크는 이런식으로 rbtree상에 존재하지 않게 된다. 하지만 on_rq 변수는 계속 1로 남아 있음에 주의하자. runqueue 관련 설정은 바뀌지 않아 정리하면 다음과 같이 된다.

|                        | Running (Current) |
|------------------------|-------------------|
| current                | O                 |
| rq->curr               | O                 |
| cfs_rq->curr           | O                 |
| cfs_rq->tasks_timeline | -                 |
| task_struct->on_rq     | O                 |
| sched_entity->on_rq    | O                 |
| task_struct->state     | TASK_RUNNING      |

## Now It's Time to Sleep
프로세스가 cpu를 점유하여 실제로 실행되고 있는 `current` 프로세스가되었다. 이제 잠을 자보자.

### Setting State and Calling Scheduler in Typical Format
[The *Current* Process State] [section-current-state] 절에서 현재 실행중인 프로세스의 상태를 알아보았으니 이제는 잠들어야 할때다. 다음의 전형적인 잠들기 코드를 보자.

	set_current_state(TASK_[WHAT_YOU_WANT]);
	if (do_i_need_to_sleep())
		schedule();

### from *TASK_RUNNABLE* state [section-from-runnable]
`state` 를 `TASK_RUNNABLE`로 설정하는 경우는 결과적으로 프로세스가 잠들지 않는다.

	set_current_state(TASK_RUNNABLE);
	if (do_i_need_to_sleep())
		schedule();

`TASK_RUNNABLE` 로 세팅하고 `schedule()` 을 의도적으로 호출하는 놈은 없겠지만 한번 알아보자.

>NOTE  
>하지만 real world 에서는 이런 일이 일어날 수 있다. `wake_up()`과의 경쟁상태 때문인데 인터럽트에 의한 컨택스트 스위칭이던 SMP에 의한 동시 실행이건 간에 깨우려는 프로세스의 `wake_up()` 이 잠들려하는 프로세스의 `schedule()` 호출전에 완료되면 해당 프로세스는 실제로 `TASK_RUNNING` 상태로 `schedule()` 을 호출하게 된다.
>[Kernel Korner - Sleeping in the Kernel] [link-kernel-kornor-sleep] 를 참고하라.

[link-kernel-kornor-sleep]: http://www.linuxjournal.com/article/8144

	kernel/sched/core.c:
	
	static void __sched __schedule(void)
	{
		...
		if (prev->state && !(preempt_count() & PREEMPT_ACTIVE)) {
			...
				deactivate_task(rq, prev, DEQUEUE_SLEEP);
				prev->on_rq = 0;
				...
			...
		}

`prev->state` 가 `TASK_RUNNALBE` 이므로 해당 구문은 실행되지 않음.

		...
		put_prev_task(rq, prev);
		next = pick_next_task(rq);
		...

`put_prev_task()` 는 결국 다음을 실행하고

	kernel/sched/fair.c:
	
	static void put_prev_entity(struct cfs_rq *cfs_rq, struct sched_entity *prev)
	{
		...
		if (prev->on_rq) {
			...
			__enqueue_entity(cfs_rq, prev);
			...
		}
		cfs_rq->curr = NULL;
	}

`prev->on_rq`가 1 이므로 다시 rbtree에 들어가게 됨.

	pick_next_taks()
		pick_next_task_fair();
			set_next_entity()
				if (se->on_rq)
					__dequeue_entity(cfs_rq, se);
				cfs_rq->curr = se;

`pick_next_task()` 에서 임의의 태스크가 선택되고 cfs_rq->curr 를 갱신.

	...
	next = pick_next_task(rq);
	if (likely(prev != next)) {
		...
		rq->curr = next;
		...
		context_switch(rq, prev, next);
	}
	...

	
`context_switch()` 가 호출되고 나면 `current`는 더이상 이전의 태스크가 아니며 이전 프로세스의 상태는 다음과 같이 된다. rbtree에 태스크가 다시 추가되었으므로 언젠가 `pick_next_task()` 에서 선택될 기회를 가지게 된다.

|                        | Running but Not Current |
|------------------------|-------------------------|
| current                | -                       |
| rq->curr               | -                       |
| cfs_rq->curr           | -                       |
| cfs_rq->tasks_timeline | O                       |
| task_struct->on_rq     | O                       |
| sched_entity->on_rq    | O                       |
| task_struct->state     | TASK_RUNNING            |

### from *TASK_INTERRUPTIBLE* or TASK_UNINTERRUPTIBLE* state
이제 프로세스가 실제로 잠드는 경우를 살펴본다.

	kernel/sched/core.c:
	
	static void __sched __schedule(void)
	{
		...
		if (prev->state && !(preempt_count() & PREEMPT_ACTIVE)) {
			...
				deactivate_task(rq, prev, DEQUEUE_SLEEP);
				prev->on_rq = 0;
				...
			...
		}

`prev->state` 가 true 이므로 해당 구문이 실행됨.

	deactivate_task()
		dequeue_task_fair()
			dequeue_entity()
				if (se != cfs_rq->curr)
					__dequeue_entity(cfs_rq, se);
				se->on_rq = 0;
    prev->on_rq = 0

이를 통해 rbtree에서 해당 태스크는 제거된 상태에서 (current 프로세스는 실제
rbtree 에 존재하지 않는다) `prev->on_rq` 를 0 으로 만듦으로써
`put_prev_task()` 에서 rbtree 에 삽입될 기회를 잃게 되어 결과적으로 잠들게
된다.

	...
	put_prev_task(rq, prev);
	next = pick_next_task(rq);
	...

`put_prev_task()` 는 결국 다음을 실행하고

	static void put_prev_entity(struct cfs_rq *cfs_rq, struct sched_entity *prev)
	{
		...
		if (prev->on_rq) {
			...
			__enqueue_entity(cfs_rq, prev);
			...
		}
		cfs_rq->curr = NULL;
	}

`prev->on_rq`가 0 이므로 해당 구문 실행안됨.  
`pick_next_task()` 부터는 [from *TASK_RUNNABLE* state] [section-from-runnable] 절 참고. 내용 동일.


|                        | Sleeping                                   |
|------------------------|--------------------------------------------|
| current                | -                                          |
| rq->curr               | -                                          |
| cfs_rq->curr           | -                                          |
| cfs_rq->tasks_timeline | -                                          |
| task_struct->on_rq     | -                                          |
| sched_entity->on_rq    | -                                          |
| task_struct->state     | TASK_INTERRUPTIBLE or TASK_UNINTERRUPTIBLE |

---

## How Process Wakes up [section-process-wakes-up]
[Process Wakes up and Becomes TASK_RUNNING] [section-process-wakes-up] 절과 내용 동일. 해당 절 참고.

---

## Sleep States and Signals [section-states-signals]

	kernel/signal.c:
	
	void signal_wake_up_state(struct task_struct *t, unsigned int state)
	{
		set_tsk_thread_flag(t, TIF_SIGPENDING);
		if (!wake_up_state(t, state | TASK_INTERRUPTIBLE))
			kick_process(t);
	}

`signal_wake_up_state()` 는 프로세스에 시그널을 보낼 때 사용되며 state 에 `TASK_WAKEKILL`/`__TASK_TRACED` 혹은 0 이 전달되며 전자의 경우 해당 플래그가 설정된 태스트들을 추가로 깨울 수 있으며 후자의 경우 
`TASK_INTERRUPTIBLE` 인 프로세스만을 깨울 수 있다.

| 태스크의 수면 상태와 중단 가능한 시그널|
| state                | signal   |
|----------------------|----------|
| TASK_INTERRUPTIBLE   | all      |
| TASK_UNINTERRUPTIBLE | x        |
| TASK_KILLABLE        | SIGKILL  |

>NOTE  
>`TASK_STOPPED` or `TASK_TRACED` 는 커널 내부의 ptrace 관련 코드에서 사용됨.  
> 추가 자료는 [TASK_KILLABLE: New process state in Linux] [link-task-killable] 참고

[link-task-killable]: http://www.ibm.com/developerworks/linux/library/l-task-killable/#author1
