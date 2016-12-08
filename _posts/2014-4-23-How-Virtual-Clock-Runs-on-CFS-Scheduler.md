---
layout: post
title: How Virtual Clock Runs on CFS Scheduler
---

## Contents

1. [Introduction][section-intro]  
2. [Nice and Priority][section-nice-prio]  
3. [Priority and Weight][section-prio-weight]  
4. [Weight and CPU Share][section-weigh-cpu-share]  
5. [Scheduling Latency][section-schedule-latency]  
6. [Time Slice and Virtual Runtime][section-timeslice-vruntime]  
6.1 [Equation of Time][section-equation-time]  
6.2 [Function for Time Slice][section-functions-timeslice]  
6.3 [Function for Virtual Runtime][section-function-vruntime]  
7. [Time Slice Expiry and _TIF_NEED_RESCHED flag][section-time-expiry]  
7.1 [Execution Time Updates for Schedule Entity][section-exec-time-update]  
7.2 [Checking Time Slice Expiry][section-checking-time-slice-expiry]  

---

## 1. Instroduction [section-intro]

CFS 에서 가중치에 따라 각 태스크가 CPU 시간을 어떻게 할당받는지 Virtual Runtime
은 어떻게 흐르는지 알아보고 언제 어떻게 태스크가 자신의 할당 시간을 채우고 선점
이 필요함을알리는지 알아본다.

>NOTE  
>Linux Version 3.13.0-rc0  

---

## 2. Nice and Priority [section-nice-prio]

    nice                                            |-20             +19|
      ^                                             v                   v
      |         +----------------....---------------+-------------------+
      |         | Real Time                         ||||||||Normal|||||||
      |         +----------------....---------------+---------+---------+
      v         ^                                   ^         ^         ^
    priority  0 |                                 99|100      |      139|
                                                              |
                                                default priority 120 (nice 0)

    include/linux/sched/rt.h:

    #define MAX_USER_RT_PRIO        100
    #define MAX_RT_PRIO             MAX_USER_RT_PRIO

    #define MAX_PRIO                (MAX_RT_PRIO + 40)
    #define DEFAULT_PRIO            (MAX_RT_PRIO + 20)
    ...

    kernel/sched/sched.h:

    /*
     * Convert user-nice values [ -20 ... 0 ... 19 ]
     * to static priority [ MAX_RT_PRIO..MAX_PRIO-1 ],
     * and back.
     */
    #define NICE_TO_PRIO(nice)      (MAX_RT_PRIO + (nice) + 20)
    #define PRIO_TO_NICE(prio)      ((prio) - MAX_RT_PRIO - 20)
    ...

nice 는 유저레벨에 노출된 값이며, priority 는 nice 와 연관된 커널 내부의
우선순위이다. 그중 priority 100 ~ 139 까지가 CFS 의 관리 대상이 되는 태스크이
다.  

---

## 3. Priority and Weight [section-prio-weight]

    kernel/sched/sched.h:

    struct load_weight {
        unsigned long weight, inv_weight;
    };

    static const int prio_to_weight[40] = {
     /* -20 */     88761,     71755,     56483,     46273,     36291,
     /* -15 */     29154,     23254,     18705,     14949,     11916,
     /* -10 */      9548,      7620,      6100,      4904,      3906,
     /*  -5 */      3121,      2501,      1991,      1586,      1277,
     /*   0 */      1024,       820,       655,       526,       423,
     /*   5 */       335,       272,       215,       172,       137,
     /*  10 */       110,        87,        70,        56,        45,
     /*  15 */        36,        29,        23,        18,        15,
    };

nice 값에 따른 weight 값을 미리 계산해둔 테이블.  
nice 0 을 weight 1024 으로 정해두고 nice 1의 차이에 있어 CPU 점유율이 약 10% 의
차이를 갖도록 `weight ~= 1024 / (1.25)^(nice)` 식에 의해 계산된 값들이다.  

    static const u32 prio_to_wmult[40] = {
     /* -20 */     48388,     59856,     76040,     92818,    118348,
     /* -15 */    147320,    184698,    229616,    287308,    360437,
     /* -10 */    449829,    563644,    704093,    875809,   1099582,
     /*  -5 */   1376151,   1717300,   2157191,   2708050,   3363326,
     /*   0 */   4194304,   5237765,   6557202,   8165337,  10153587,
     /*   5 */  12820798,  15790321,  19976592,  24970740,  31350126,
     /*  10 */  39045157,  49367440,  61356676,  76695844,  95443717,
     /*  15 */ 119304647, 148102320, 186737708, 238609294, 286331153,
    };

이 테이블은 inv_weight 에 사용되는 테이블로 weight 값으로 나누기를 할 때,
나누기대신 곱하기를 사용할 수 있도록 `2^32 / weight` 를 미리 계산해둔 값이다.  
2^32 의 숫자에 나누는 이유는 충분히 큰수에 나누어 소수점으로 버리는 수가 적도록
하기 위함이다.  

---

## 4. Weight and CPU Share [section-weigh-cpu-share]

    cpu_share = weight_of_a_process / weight_of_all_runnable_processes_on_a_rq

예를 들어 다음과 같은 프로세스들이 있다고 할 때  

| Process   | Nice  | Weight    | CPU Share             |
|-----------|-------|-----------|-----------------------|
| A         | -2    | 1586      | 29.57 % (1586 / 5362) |
| B         | -1    | 1277      | 23.81 %               |
| C         |  0    | 1024      | 19.09 %               |
| D         | +1    | 820       | 15.29 %               |
| E         | +2    | 655       | 12.21 %               |

위와 같이 각 프로세스의 CPU 점유율을 계산할 수 있다.  

---

## 5. Scheduling Latency [section-schedule-latency]

    kernel/sched/fair.c:

    static u64 __sched_period(unsigned long nr_running)
    {
        u64 period = sysctl_sched_latency;
        unsigned long nr_latency = sched_nr_latency;

        if (unlikely(nr_running > nr_latency)) {
                period = sysctl_sched_min_granularity;
                period *= nr_running;
        }

        return period;
    }

간략히 표현하면  

    nr_running <= sched_nr_latency ? sysctl_sched_latency
                                   : sysctl_sched_min_granularity * nr_running

sysctl_sched_latency            : time all runnable tasks are scheduled at
                                  least once.
                                  default, 6 ms  
sysctl_sched_min_granularity    : minimum time before preempted.
                                  default, 0.75 ms  
sched_nr_latency                : nr_latency / min_granularity.
                                  default, 8  

>NOTE  
sysctl_sched_latency and sysctl_sched_min_gradularity are tunable.  

---

## 6. Time Slice and Virtual Runtime [section-timeslice-vruntime]
### 6.1 Equation of Time [section-equation-time]

weight 를 가진 어떤 프로세스가 실제 할당받는 CPU 시간은 다음으로 나타낼 수
있고  

    time_slice = weight / weight_of_rq * (min_granularity * nr_running)

>NOTE  
sched_nr_latency 를 무시하고 일반적으로 나타낸 공식  

할당 받은 CPU 시간을 CFS의 가상 시간으로 나타내기 위해 다음 공식을 사용한다.  

    vruntime_of_a_process = time * weight_of_nice_0 / pruntime_of_a_process

같은 물리적인 시간에 대해 weight 가 높은 프로세스의 가상시간은 weight 가 낮은
프로세스에 대비해 상대적으로 느리게 흐르게 되어 같은 가상 시간을 채우기 위해
보다 많은 물리적인 시간이 필요해지게 된다. 이 러한 특성으로 인해 weight가 높은
프로세스가 보다 많은 CPU 시간을 사용하게 된다.  

| Process   | Nice  | Wp/Wr     | Time Slice    | W0/Wp     | VRuntime  |
|-----------|-------|-----------|---------------|-----------|-----------|
| A         | -2    | 0.2957    | 1.7742 ms     | 0.6456    | 1.1454    |
| B         | -1    | 0.2381    | 1.4286        | 0.8018    | 1.1454    |
| C         |  0    | 0.1909    | 1.1454        | 1         | 1.1454    |
| D         | +1    | 0.1529    | 0.9174        | 1.2487    | 1.1455    |
| E         | +2    | 0.1221    | 0.7326        | 1.5633    | 1.1452    |

>NOTE  
Wp              : weight of a process  
Wr              : weight of a runqueue  
W0              : weight of nice 0, namely, 1024  
VRuntime        : Time Slice * (NICE0 / Weight)  
sched_latency   : 6 ms  
min_granularity : 0.75 ms  

예로 든 위의 표와 같이 weight 가 높은 프로세스는 실제적으로 CPU 시간을 더 많이
점유하지만 virtual runtime 은 거의 같게 유지된다.  

>NOTE  
>프로세스 E 의 경우 min_graunlarity 0.75 보다 적은 time slice 가 할당되었기
>때문에 계산된 time slice 대신에 0.75 만큼 실행되지만 초과된 시간만큼 보다 큰
>virtual runtime이 반영되어 rq의 훨씬 뒤쪽에 삽입될 것이다.  

### 6.2 Function for Time Slice [section-functions-timeslice]

| Function              | Time              |
|-----------------------|-------------------|
| **calc_delta_mine()** | **time slice**    |
| calc_delta_fair()     | virtual runtime   |

weight 를 가진 프로세스에 대한 time slice 는 다음 공식으로 나타낼 수 있으며  

    time_slice = (min_granularity * nr_running) * (weight / weight_of_rq)

`calc_delta_mine((min_granularity * nr_running), weigh, weigh_of_rq)` 로
구할 수 있다.  

    kernel/sched/fair.c:

    #if BITS_PER_LONG == 32
    # define WMULT_CONST    (~0UL)
    #else
    # define WMULT_CONST    (1UL << 32)
    #endif

    #define WMULT_SHIFT     32

    #define SRR(x, y) (((x) + (1UL << ((y) - 1))) >> (y))

    static unsigned long
    calc_delta_mine(unsigned long delta_exec, unsigned long weight,
                    struct load_weight *lw)
    {
        u64 tmp;

        if (likely(weight > (1UL << SCHED_LOAD_RESOLUTION)))
            tmp = (u64)delta_exec * scale_load_down(weight);
        else
             tmp = (u64)delta_exec;

        if (!lw->inv_weight) {
            unsigned long w = scale_load_down(lw->weight);

            if (BITS_PER_LONG > 32 && unlikely(w >= WMULT_CONST))
                lw->inv_weight = 1;
            else if (unlikely(!w))
                lw->inv_weight = WMULT_CONST;
            else
                lw->inv_weight = WMULT_CONST / w;
        }

        if (unlikely(tmp > WMULT_CONST))
            tmp = SRR(SRR(tmp, WMULT_SHIFT/2) * lw->inv_weight, WMULT_SHIFT/2);
        else
            tmp = SRR(tmp * lw->inv_weight, WMULT_SHIFT);

        return (unsigned long)min(tmp, (u64)(unsigned long)LONG_MAX);
    }

(delta_exec * weight) / lw->weight  

### 6.3 Function for Virtual Runtime [section-function-vruntime]

| Function              | Time                  |
|-----------------------|-----------------------|
| calc_delta_mine()     | time slice            |
| **calc_delta_fair()** | **virtual runtime**   |

주어진 시간에 대한 virtual time 은 다음 공식으로 나타낼 수 있으며  

    vruntime_of_a_process = time * weight_of_nice_0 / time_slice_of_a_process

`cal_delta_fair()` 는 `calc_delta_mine(time_slice, 1024, weigh)` 으로 구현된다.
  
    kernel/sched/fair.c:

    static inline unsigned long
    calc_delta_fair(unsigned long delta, struct sched_entity *se)
    {
        if (unlikely(se->load.weight != NICE_0_LOAD))
            delta = calc_delta_mine(delta, NICE_0_LOAD, &se->load);

        return delta;
    }

(delta * 1024) / se->load->weight

---

## 7. Time Slice Expiry and _TIF_NEED_RESCHED flag [section-time-expiry]

    scheduler_tick()
        update_rq_clock()
        task_tick_fair()
            entity_tick()
                update_curr()
                    __update_curr()
                check_preempt_tick()
                    resched_task()
                        set_tsk_need_resched()

irq 에서 타이머 핸들러가 실행되고 필요에 의해 set_tsk_need_resched() 가 호출
되기 까지의 흐름이다. `set_tsk_need_resched()` 가 `_TIF_NEED_RESCHED` 플래그를
설정하면 irq 모드에서 유저/커널모드로 복귀하기 직전에 다른 태스크가 현재 태스크
를 선점하도록 스케쥴러가 호출된다.

### 7.1 Execution Time Updates for Schedule Entity [section-exec-time-update]

    include/linux/sched.h:

    struct sched_entity {
        ...
        u64 exec_start;
        u64 sum_ecec_runtime;
        u64 vruntime;
        u64 prev_sum_exec_runtime;
        ...
    };

스케쥴링의 기본 단위내에 시간관련 멤버변수들.  

    kernel/sched/fair.c:

    static void update_curr(struct cfs_rq *cfs_rq)
    {
        u64 now = rq_clock_task(rq_of(cfs_rq));

`update_rq_clock()` 에서 갱신된 현재 시간을 읽는다.  

        ...

        delta_exec = (unsigned long)(now - curr->exec_start);
        ...

지난번 `update_curr()` 실행 이후 지금까지의 지나간 시간 간격을 구한다.

        __update_curr(cfs_rq, curr, delta_exec);
        curr->exec_start = now;
        ...
    }

`__update_curr()` 를 이용하여 `delta_exec` 현재 실행중인 entity의 시간 정보를
갱신한 후, exec_start 를 now 값으로 갱신하여 다음번 `update_curr()` 에서 delta
값을 구하기 위한 준비를 한다.  

    static inline void
    __update_curr(struct cfs_rq *cfs_rq, struct sched_entity *curr,
                  unsigned long delta_exec)
    {
        curr->sum_exec_runtime += delta_exec;
        ...

caller 에서 구해진 delta_exec 값을 `sum_exec_runtime` 변수에 더함으로써 총 실행
시간을 갱신한다.

        delta_exec_weighted = calc_delta_fair(delta_exec, curr);

        curr->vruntime += delta_exec_weighted;

delta_exec 시간값을 virtual runtime으로 변환 하기 위해 `calc_delta_fair()`를
사용한다. 가중치가 적용된 가상시간을 vruntime에 더하여 총 실행 시간을 갱신한다.
6.3 절참고.  

        update_min_vruntime(cfs_rq);
    }

해당 rq의 기준 vruntime이라고 볼 수 있는 cfs_rq->min_vruntime 을 갱신한다. 이
값은 se 를 rq 간 이동(삽입/제거)시킬 때 se->vruntime 에 대하여 더하거나 빼짐으
로써 se 가 이동할 rq 에 놓을 시간적 위치를 이전 rq의 시간적 위치와 상대적으로
동일한 시간 위치에 놓이도록하기 위해 사용된다.

### 7.2 Checking Time Slice Expiry [section-checking-time-slice-expiry]

    kernel/sched/fair.c:

    static void
    check_preempt_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr)
    {
        ...
        ideal_runtime = sched_slice(cfs_rq, curr);

`sched_slice()` 를 통해 현재 엔티티가 실행해야하는 이상적인 CPU 점유 시간을
구한다.  

        delta_exec = curr->sum_exec_runtime - curr->prev_sum_exec_runtime;

`prev_sum_exec_runtime`은 현재 엔티티가 실행되기 직전 `sum_exec_runtime` 으로
설정해놓은 것으로 매 틱마다 갱신되는 `sum_exec_runtime` 과 비교함으로써 총 실행
시간의 차를 구할 수 있다.  

        if (delta_exec > ideal_runtime) {
            resched_task(rq_of(cfs_rq)->curr);

현재 엔티티가 가중치에 따라 실제로 점유해야할 CPU 시간과 총 실행한 시간과 비교
하여 이상적인 점유 시간을 넘어 실행하였다면 `resched_task()` 를 호출하여 현재
엔티티가 스케쥴링이 필요함을 알린다.  

            clear_buddies(cfs_rq, curr);
            return;
        }

        if (delta_exec < sysctl_sched_min_granularity)
            return;

        se = __pick_first_entity(cfs_rq);
        delta = curr->vruntime - se->vruntime;

        if (delta < 0)
            return;

        if (delta > ideal_runtime)
            resched_task(rq_of(cfs_rq)->curr);
    }

이상적인 CPU 점유시간을 모두 사용하지 않은 경우에도 rq 에 있는 가장 작은
vruntime 을 갖는 엔티티와의 vruntime 시간차가 ideal_runtime을 넘는 경우에도
`resched_task()` 를 호출한다.  
