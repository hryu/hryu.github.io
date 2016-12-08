---
layout: post
title: Mininum Virtual Runtime of CFS Run Queue (min_vruntime)
---

## Contents

1. [Introduction][section-introduction]  
2. [Where and How min_vruntime Updated][section-update-vruntime]  
3. [min_vruntime and Run Queue Migration][section-rq-migration]  
4. [min_vruntime and Sleep and Wake Up][section-sleep-wakeup]  
5. [min_vruntime and New Task Wake Up][section-new-task-wakeup]  

---

## 1. Instroduction [section-introduction]

cfs_rq->min_vruntime 은 해당 rq 의 기준점이 되는 vruntime 으로서 se 의 시간
통계를 갱신할 때마다 따라서 계속적으로 갱신 (증가) 되며 se 를 rq 간 이동시킬 때
se->vruntime 에 빼고 더함으로써 이전 rq 에서 놓였던 시간위치를 이동할 rq 에서도
유지할 수 있도록 한다. 또한 sleep (혹은 갓 fork 된) 상태였던 프로세스들을 rq 에
다시 삽입할때에도 참조되어 적절한 위치에 삽입될 수 있도록 한다.

>NOTE  
>commit 88ec22d3edb72b261f8628226cd543589a6d5e1b  
>sched: Remove the cfs_rq dependency from set_task_cpu()  
>참고.  
>  
>Kernel Version 3.13.0-rc8  

## 2. Where and How min_vruntime Updated [section-update-vruntime]

cfs_rq->min_vruntime 은 1. update_curr()로 cfs_rq->curr 의 시간값을 갱신할 때나
2. dequeue_entity() 로 se 를 rq 로부터 제거하였을 때 갱신된다.

    static void update_min_vruntime(struct cfs_rq *cfs_rq)
    {
        u64 vruntime = cfs_rq->min_vruntime;

        if (cfs_rq->curr)
            vruntime = cfs_rq->curr->vruntime;

        if (cfs_rq->rb_leftmost) {
            struct sched_entity *se = rb_entry(cfs_rq->rb_leftmost,
                                               struct sched_entity,
                                               run_node);

            if (!cfs_rq->curr)
                vruntime = se->vruntime;
            else
                vruntime = min_vruntime(vruntime, se->vruntime);
        }

curr 의 vruntime 과 rq 의 leftmost 중 보다 작은 vruntime 이 선택되고
(curr 는 rq 의 타임라인에 존재하지 않으며 dequeue 된 후 실행되지만 `se->on_rq`,
`p->on_rq` 는 계속 1 로 유지됨에 유의하자.)

        /* ensure we never gain time by being placed backwards. */
        cfs_rq->min_vruntime = max_vruntime(cfs_rq->min_vruntime, vruntime);
    }

그렇게 선택된 vruntime 과 rq 의 min_vruntime 중 보다 큰 값으로 rq 의 기존
min_vruntime 을 갱신하여 rq 의 min_vruntime 이 계속적으로 증가되도록 한다.
즉 해당 rq 에서 curr 포함, 가장 작은 vruntime 이며 다음식으로 간단히 나타낼 수
있다.

    cfs_rq->min_vruntime = MAX(cfs_rq->min_vruntime,
                               MIN(cfs_rq->curr->.vruntime,
                                   cfs_rq->rb_left_most->vruntime))

## 3. min_vruntime and Run Queue Migration [section-rq-migration]

rq 간에 상이한 min_vruntime (일종의 0 이 되는 기준점이라 볼 수 있다.) 때문에 se
를 rq 간 이동시키기 위해서는 rq 에 독립적이면서 runnable 한 모든 se 들을 상대적
인 시간적 거리만을 가지고 다룰 필요가 있다. 그렇기 위해 se 를 rq 에서 제거할때
se->vruntime 으로 부터 해당 rq->min_vruntime 을 뺌으로써 se->vruntime 을 rq
독립적인 상태로 변환한 후, 삽입할 rq 의 min_vruntime 을 더함으로써 이동할 rq 에
서도 이전 rq 에서 놓였던 시간위치를 유지할 수 있도록 한다. 다음의 예를 보자.

    |
    |-----> | <----- OFFSET ------> | ...
    |       v                       v
    |       cfs_rq_1->min_vruntime  se_1->vruntime
    |
    |-> | <---------------------------------------> | ...
    |   v                                           v
    |   cfs_rq_2->min_vruntime                      se_2->vruntime
    |
    | <-----------> | ...
    |               v
    v               se_3->vruntime
    |cfs_rq_3->min_vruntime
    |

그림과 같이 rq 들과 se 들이 분포한다고 가정하고 se_1 을 cfs_rq_1 에서 cfs_rq_2
로 옮겨보자.

    |
    | <----- OFFSET ------> |
    |                       v
    |                       se_1->vruntime_relative
    |                           (se_1->vruntime - cfs_rq_1->min_vruntime)
    |
    |-> | <---------------------------------------> | ...
    |   v                                           v
    |   cfs_rq_2->min_vruntime                      se_2->vruntime
    |

먼저 se_1->vruntime 에서 cfs_rq_1->min_vruntime 을 빼서 se_1->vruntime 을 rq 에
독립적인 상태로 만든 후,

    |
    |-> | <----- OFFSET ------> | <---------------> | ...
    |   v                       |                   v
    |   cfs_rq_2->min_vruntime  v                   se_2->vruntime
    |                           se_1->vruntime + cfs_rq_2->min_vruntime

se_1->vruntime 에 이동할 cfs_rq_2->min_vruntime 을 더하여 해당 rq 의 타임라인
에서 se_1이 삽입될 위치를 정하게 된다. (OFFSET 이 유지되고 있음에 유의하자)

    dequeue_entity(flags)
        if (!(flags & DEQUEUE_SLEEP))
            se->vruntime -= cfs_rq->min_vruntime;

`dequeue_entity(0)` 으로 se->vruntime 을 rq 독립적으로 만들 수 있다.

    enqueue_entity(flags)
        if (!(flags & ENQUEUE_WAKEUP) || flags & ENQUEUE_WAKING)
            se->vruntime += cfs_rq->min_vruntime;

`enqueue_entity(0)` 으로 se->vruntime 을 다시 특정 rq 의 타임라인에 종속적으로
만든다.

    migrate_tasks()     migration_cpu_stop()
        |                   |
        +-------+-----------+
                |
                v
            __migrate_task()
                if (on_rq)
                    dequeue_task(rq_src, 0)
                        dequeue_entity(0)
                            se->vruntime -= cfs_rq->min_vruntime;
                set_task_cpu()
                if (on_rq)
                    enqueue_task(rq_dest, 0)
                        enqueue_entity(0)
                            se->vruntime += cfs_rq->min_vruntime;

위와 같이 vruntime 을 조작하여 se->vruntime 을 상대적 시간값으로 변화하는 경우
에는 1. rq 간 이동이외에도 2. priority 변경이나 3. 스케쥴러정책변경과 같은 다른
제어흐름도 있다.

## 4. min_vruntime and Sleep and Wake Up [section-sleep-wakeup]

rq 를 이동할때에는 enqueue 시에 min_vruntime 을 빼고 dequeue 시에 min_vruntime
을 더해 이동한 rq 에서도 min_vruntime 으로 부터 일정한 offset 을 유지하도록하
는데 반해 sleep/wakeup 시에 dequeue 에서는 min_vruntime 을 보정하지 않으며 이로
인해 삽입되는 rq 에서 이전 rq->min_vruntime 과의 offset 이 유지되지 않고
`place_entity()` 에서 rq->min_vruntime 과 비교 및 추가 연산을 통해 새로운 자리
에 삽입된다.

### 4.1 Dequeue on Sleep [section-dequeue-sleep]

    __schedule()
        deactivate_task(DEQUEUE_SLEEP)
            dequeue_task(DEQUEUE_SLEEP)
                dequeue_entity(DEQUEUE_SLEEP)
                    update_curr()
                        curr->vruntime += calc_delta_fair()
                        update_min_vruntime()
                    if (se != cfs_rq->curr)
                        __dequeue_entity(cfs_rq, se)
                    if (!(flags & DEQUEUE_SLEEP))
                        se->vruntime -= cfs_rq->min_vruntime;

`DEQUEUE_SLEEP` 을 인자로 넘겨 vruntime 을 보정하지 않도록 한다. 이것은 실행되
던 rq 의 min_vruntime 을 보존하는 효과를 갖는다.

### 4.2 Enqueue on Wake Up [section-enqueue-wakeup]

    try_to_wake_up()
        p->sched_class->task_waking() == task_waking_fair()
            min_vruntime = cfs_rq->min_vruntime
            se->vruntime -= min_vruntime
        ttwu_queue()
            ttwu_do_acticate()
                ttwu_activate(ENQUEUE_WAKEUP | ENQUEUE_WAKING)
                    activate_task(ENQUEUE_WAKEUP | ENQUEUE_WAKING)
                        enqueue_task(ENQUEUE_WAKEUP | ENQUEUE_WAKING)
                            enqueue_entity(ENQUEUE_WAKEUP | ENQUEUE_WAKING)
                                if (!(flags & ENQUEUE_WAKEUP) ||
                                     (flags & ENQUEUE_WAKING))
                                    se->vruntime += cfs_rq->min_vruntime
                                update_curr()
                                    curr->vruntime += calc_delta_fair()
                                    update_min_vruntime()
                                if (flags & ENQUEUE_WAKEUP)
                                    place_entity(cfs_rq, se, 0)
                                if (se != cfs_rq->curr)
                                    __enqueue_entity()

여기서 cfs_rq->min_vruntime 은 se 를 실행할때마다 갱신 (증가) 된 상태이며
se->vruntime 에서 현재 min_vruntime 을 뺀후, 다시 더한다는 것은 다음과 같은 식
으로도 볼 수 있다.

    se->vruntime = (se->vruntime_relative + rq->min_vruntime_OLD)
                        - rq->min_vruntime_NOW + rq->min_vruntime_NOW

`se->vruntime` 에 같은 값을 더하고 다시 뺐으니 결국 원래 sleep 하던 시점의
vruntime 이 그대로 사용된다.
이 값은 `place_entity()` 함수에서 경우에 따라 다시 갱신되어 enqueue 시 타임라인
의 올바른 자리에 se 가 삽입될 수 있도록한다.

    static void
    place_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int initial)
    {
        u64 vruntime = cfs_rq->min_vruntime;

        if (initial && sched_feat(START_DEBIT))
            vruntime += sched_vslice(cfs_rq, se);

새로이 fork 된 프로세스는 경우에 따라 vruntime 에 총 time slice 중 se 의 weight
에 따른 slice 값인 vslice 을 더하여 보다 뒤에 삽입되도록한다.

        if (!initial) {
            unsigned long thresh = sysctl_sched_latency;
            if (sched_feat(GENTLE_FAIR_SLEEPERS))
                thresh >>= 1;

            vruntime -= thresh;
        }

sleep 하던 se 의 경우 `sys_sched_latency` 만큼 빼서 이번 스케쥴턴 안에 삽입되
도록한다.

        se->vruntime = max_vruntime(se->vruntime, vruntime);
    }

보다 큰값으로 `se->vruntime` 값을 갱신한다.

>NOTE  
> sysctl_sched_latency :  
>  time all runnable tasks are scheduled at least once.  
>  default, 6 ms  

## 5. min_vruntime and New Task Wake Up [section-new-task-wakeup]

    do_fork()
        copy_process()
            sched_fork()
                task_fork_fair()
                    se->vruntime = curr->vruntime
                    place_entity()
                    se->vruntime -= cfs_rq->min_vruntime
        wake_up_new_task()
            activate_task(0)
                enqueue_task(0)
                    enqueue_entity(0)
                        se->vruntime += cfs_rq->min_vruntime
                        update_curr()
                            update_min_vruntime()
                        __enqueue_entity()

min_vruntime 으로 부터의 offset 이 유지된다는 점에서 3.절 rq 이동과 내용이 대동
소이하다.
