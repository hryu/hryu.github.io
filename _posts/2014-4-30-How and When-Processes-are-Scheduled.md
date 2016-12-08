---
layout: post
title: How and When Processes are Scheduled
---

## Contents

1. [Introduction][section-introduction]  
2. [Setting TIF_NEED_RESCHED flag][section-setting-tif-flag]  
2.1. [Mark Task to be Scheduled][section-mark-task-sched]  
2.2. [Tick Based Activities][section-tick-base-activity]  
2.2.1. [Per Runqueue High Resolution Timer][section-per-rq-hrtimer]  
2.2.2. [Scheduler Timer for Tick Emulation][section-sched-timer]  
2.3. [Process Waking up Activity][section-proc-wakeup]  
2.3.1. [Fork and Wake Up a New Process][section-fork-proc]  
2.3.2. [Wake Up a Sleeping Process][section-wakeup-proc]  
2.4. [Process Migration Activity][section-migrate-proc]  
2.4.1. [Execute Binary][section-exec-bin]  
2.4.2. [Hot-plug Down CPU][section-hotplug-down-cpu]  
2.4.3. [Modify CPU Allowed Mask][section-modify-cpu-allowed]  
2.4.4. [Load Balancing][section-load-balance]  
2.5. [Priority or Scheduler Class Change Activity]
     [section-change-prior-sched]  
2.5.1. [Setting Nice Value][section-set-nice]  
2.5.2. [Change Scheduler Classes][section-change-sched-class]  
2.5.3. [Priority Change When Locking/Unlocking RTMutex][section-rt-mutex]  
2.5.4. [Normalize RT Tasks via Magic Key n][section-magic-key-n]  
2.6. [Bandwidth Control Activity][section-bandwidth-ctrl]  
2.7. [Various Other Sites][section-other-sites]  
3. [Checking TIF_NEED_RESCHED flag and Calling Scheduler]
   [section-calling-scheduler]  
3.1. [Process and Processor Modes][section-proc-modes]  
3.1.1. [User Mode and Exceptions][section-usr-mode-exception]  
3.1.2. [Kernel Mode and Exceptions][section-kernel-mode-exception]  
3.2. [Return to User Mode][section-return-user-mode]  
3.2.1. [Return from System Call][section-return-user-syscall]  
3.2.2. [Return from IRQ][section-return-user-irq]  
3.2.3. [Return from Other Exceptions][section-return-user-other-excpnt]  
3.3. [Return to Kernel Mode][section-return-kernel-mode]  
3.3.1. [Return from IRQ][section-return-kernel-irq]  
3.3.2. [Kernel Mode Preemption][section-kernel-preemptiom]

---

## 1. Introduction [section-introduction]

언제 어떻게 프로세스가 스케쥴되는가?  
`TIF_NEED_RESCHED` 가 프로세스 상태 플래그에 설정되고 이후 몇가지 지점에서
해당 플래그를 판단하여 스케쥴링을 호출하는데 이러한 흐름을 살펴본다.  

> **NOTE**  
> 3.15.0-rc2 (Shuffling Zombie Juror)  
>
> 본 문서는 ARM port, CFS, high resolution timer, dynamic tick 에 의존적이다.  
> 밀접한 관련이 있는 time management 는 다루지 않는다.  
> 자발적 스케쥴링은 다루지 않는다.  
> sleepable lock 획득을 위한 스케쥴링은 자발적이라고 간주한다.  

---

## 2. Setting TIF_NEED_RESCHED [section-setting-tif-flag]

    arch/arm/include/asm/thread_info.h:

    struct thread_info {
        unsigned long flags
        ...
    };

    #define TIF_NEED_RESCHED        1
    ...
    #define _TIF_NEED_RESCHED       (1 << TIF_NEED_RESCHED)
    ...

`TIF_NEED_RESCHED` 는 `flags`에 세팅되는 값이며 다양한 매크로들을 통해
set/clear/test 되어 해당 값이 셋되었을 때 해당 프로세스가 스케쥴링이 필요함을
나타낸다.

`_TIF_NEED_RESCHED` 는  
1. `do_work_pending()` 에서 사용되어 시스템콜과 인터럽트에서 유저모드로의 복귀
   직전이나  
2. 커널 모드 동작 중, `preempt_enable()` 등으로 선점이 활성화되는 순간 혹은
   irq에서 커널 모드로 복귀직전에 스케쥴러를 호출할지 결정할 때 사용된다.  

`set_tsk_need_resched()` 함수는 태스크의 플래그에 `TIF_NEED_RESCHED` 플래그를
직접설정하는 함수이다.  

### 2.1. Mark Task to be Scheduled [section-mark-task-sched]

    kernel/sched/core.c:

    void resched_task(struct task_struct *p)
    {
        int cpu;

        if (test_tsk_need_resched(p))
                return;

        set_tsk_need_resched(p);

        cpu = task_cpu(p);
        if (cpu == smp_processor_id()) {
            set_preempt_need_resched();
            return;
        }

        /* NEED_RESCHED must be visible before we test polling */
        smp_mb();
        if (!tsk_is_polling(p))
            smp_send_reschedule(cpu);
    }

`set_tsk_need_resched()` 를 통하여 `TIF_NEED_RESCHED` 를 설정하고  
1. 태스크가 현재 cpu 에서 실행 중이면 그냥 이전 문맥 복귀시 스케쥴될 수 있도록
   그냥 리턴하고  
2. 다른 cpu 에서 실행중이면 IPI 를 보내 해당 cpu 에서 스케쥴링을 즉시
   시도한다.  
현재 cpu 에서 실행중이면 그냥 리턴하는데 이 경우는 나중에 언급할 현재 함수가
실행중인 예외/시스템콜/irq 등의 문맥에서 복귀시 스케쥴링 된다.

    check_preempt_curr(rq, p, flags)
        check_preempt_wakeup(rq, p, wake_flags)
            curr = rq->curr
            ...
            resched_task(curr)

`check_preempt_curr()` 는 다양한 경우에 호출되어 `current` 프로세스를 `p`
프로세스로 선점해야할 필요가 있을 경우에 호출된다.  
`resched_task()` 를 호출하기 전에 몇가지 조건 검사를 실행한다.

### 2.2. Tick Based Activity [section-tick-base-activity]

    -- HARDIRQ Context --

    clock_event_device->event_handler(evt)

    hrtimer_interrupt(evt)
        for (i = 0; i < HRTIMER_MAX_CLOCK_BASES; i++)
            while ((node = timerqueue_getnext(&base->active)))
                __run_hrtimer(timer, &basenow)
                    timer->function()

high resolution hard interrupt context 에서 타이머 핸드러가 호출된다.

#### 2.2.1. Per Runqueue High Resolution Timer [section-per-rq-hrtimer]

    hrtick(timer)
        update_rq_clock(rq)
        rq->curr->sched_class->task_tick(rq, rq->curr, 1)
            entity_tick(cfs_rq, se, queued)
                update_curr(cfs_rq)
                if (queued)
                    resched_task(rq_of(cfs_rq)->curr)
                    return
        return HRTIMER_NORESTART

`hrtick()` 은 `hrtick_timer` 의 핸들러인데 `hrtick_timer` 는 `struct rq` 에
있으며 `sched_init()`의 `init_rq_hrtick()` 에서 초기화 되어 이후
`pick_next_task()` 와 `[enqueue/dequeue]_task()` 가 불릴때마다 만료 시간이
갱신된다.

#### 2.2.2. Scheduler Timer for Tick Emulation [section-sched-timer]

    tick_sched_timer(timer)
        regs = get_irq_regs()
        now = ktime_get()
        tick_sched_do_timer(now)
        if (regs)
            tick_sched_handle(ts, regs)
                update_process_times(user_mode(regs))
                    scheduler_tick()
                        update_rq_clock(rq)
                        curr->sched_class->task_tick(rq, curr, 0)
                            entity_tick(cfs_rq, se, queued)
                                update_curr(cfs_rq)
                                if (!sched_feat(DOUBLE_TICK) &&
                                    hrtimer_active(&rq->hrtick_timer))
                                    return
                                if (cfs_rq->nr_running > 1)
                                    check_preempt_tick(cfs_rq, curr)

        hrtimer_forward(timer, now, tick_period)
        return HRTIMER_RESTART

check_preempt_tick() 는 현재 실행하고 있는 엔티티가 이상적인 CPU 시간을 모두
사용한 경우 `resched_task()` 를 호출한다.

> NOTE  
> How Virtual Clock Runs on CFS Scheduler.md  
>  7.2 Checking Time Slice Expiry  

`tick_sched_timer()` 은 `tick_cpu_sched->sched_timer` 의 핸들러이며
`tick_cpu_sched` 는 per-cpu 이고 periodic tick emulation 을 위해 사용된다.

    hrtimer_switch_to_hres()
        tick_setup_sched_timer()
            tick_switch_to_oneshot(hrtimer_interrupt)
                evt->event_handler = handler
            tick_setup_sched_timer()
                ts = &__get_cpu_var(tick_cpu_sched)
                hrtimer_init(&ts->sched_timer, CLOCK_MONOTONIC,
                             HRTIMER_MODE_ABS)
                ts->sched_timer.function = tick_sched_timer

periodic tick emulation 초기화는 위와 같다.

### 2.3. Process Waking up Activity [section-proc-wakeup]
#### 2.3.1. Fork and Wake Up a New Process [section-fork-proc]

    do_fork(clone_flags, stack_start, stack_size,
            *parent_tidptr, *child_tidptr)
        copy_process(clone_flags, stack_start, stack_size, child_tidptr,
                     NULL, trace)
            sched_fork(clone_flags, p)
            copy_thread(clone_flags, stack_start, stack_size, p)
                thread->cpu_context.pc = ret_from_fork
        wake_up_new_task(p)
            activate_task(rp, p, 0)
                enqueue_task(rq, p, flags)
            check_preempt_curr(rq, p, WF_FORK)

`check_preempt_curr()`에서 경우에 따라 `TIF_NEED_RESCHED`이 세팅되며 이후
어디선가 스케쥴러가 호출되고 새로운 프로세스가 선택되었을때 컨택스트 스위칭을
할때,

    __schedule()
        context_switch(rq, prev, next)
            switch_to(prev, next, prev)

`cpu_context.pc` 의 값이 복구되며 `ret_from_fork` 호출이 호출된다.

    ret_from_fork()
        schedule_tail(prev)

자식 프로세스는 `schedule()` 을 호출한적이 없으므로 `schedule_tail()`을 통해
수동으로 `__schedule()` 마지막 부분을 호출해야한다.

#### 2.3.2. Wake Up a Sleeping Process [section-wakeup-proc]

    try_to_wake_up(p, state, wake_flags)
        ttwu_queue(p, cpu)
            ttwu_do_activate(rq, p, 0)
                ttwu_activate(rq, p, ENQUEUE_WAKEUP | ENQUEUE_WAKING)
                    activate_task(rq, p, flags)
                        enqueue_task(rq, p, flags)
                ttwu_do_wakeup(rq, p, wake_flags)
                    check_preempt_curr(rq, p, wake_flags)
                    p->state = TASK_RUNNING

`wake_up_new_task()`와 마찬가지로 `check_preempt_curr()` 에서 경우에 따라
`TIF_NEED_RESCHED`이 세팅된다. 현재 프로세스는 이후 스케쥴러가 호출되었을때
다른 프로세스로 대체된다.

### 2.4. Process Migration Activity [section-migrate-proc]
#### 2.4.1. Execute Binary [section-exec-bin]

    do_execve(filename, *__argv, *__envp)
        do_execve_common(filename, argv, envp)
            sched_exec()
                dest_cpu = p->sched_class->select_task_rq(p,
                                                          task_cpu(p),
                                                          SD_BALANCE_EXEC, 0)
                if (dest_cpu == smp_processor_id())
                    goto unlock;
                if (likely(cpu_active(dest_cpu)))
                    arg = { p, dest_cpu }
                    stop_one_cpu(task_cpu(p), migration_cpu_stop, &arg)
                    return

    migration_cpu_stop(*data)
        *arg = data
        __migrate_task(arg->task, raw_smp_processor_id(), arg->dest_cpu)
            rq_src = cpu_rq(src_cpu)
            rq_dest = cpu_rq(dest_cpu)
            ...
            if (p->on_rq)
                dequeue_task(rq_src, p, 0)
                set_task_cpu(p, dest_cpu)
                enqueue_task(rq_dest, p, 0)
                check_preempt_curr(rq_dest, p, 0)

프로세스가 새로운 바이너리로 실행을 바꿀때 로드밸런싱 차원에서 프로세스들을
목적 cpu 로 옮기면서 `check_preempt_curr()` 를 호출한다.

#### 2.4.2. Hot-plug Down CPU [section-hotplug-down-cpu]

    _cpu_down(cpu, tasks_frozen)
        hcpu = (void *)(long)cpu;
        mod = tasks_frozen ? CPU_TASKS_FROZEN : 0
        tcd_param = mod, hcpu
        err = __stop_machine(take_cpu_down, &tcd_param, cpumask_of(cpu))

     take_cpu_down(*_param)
        err = __cpu_disable()
        ...
        cpu_notify(CPU_DYING | param->mod, param->hcpu)

    migration_call(*nfb, action, *hcpu)
        cpu = (long)hcpu
        switch (action & ~CPU_TASKS_FROZEN)
        ...
        case CPU_DYING:
            ...
            migrate_tasks(cpu)
                *rq = cpu_rq(dead_cpu)
                for ( ; ; )
                    if (rq->nr_running == 1)
                        break
                    next = pick_next_task(rq, &fake_task)
                    next->sched_class->put_prev_task(rq, next)
                    dest_cpu = select_fallback_rq(dead_cpu, next)
                    __migrate_task(next, dead_cpu, dest_cpu)
                        ...
        ...

cpu 를 다운시키고 해당 cpu 에서 실행중이던 프로세스를 다른 cpu 로 옮겨
실행하려고 할때 프로세스들을 목적지 cpu 에 옮기고 `check_preempt_curr()` 를
호출한다.

>NOTE  
>`__migrate_task(next, dead_cpu, dest_cpu)` 함수는 2.4.1. Execute Binary 참고  

#### 2.4.3. Modify CPU Allowed Mask [section-modify-cpu-allowed]

	set_cpus_allowed_ptr(p, *new_mask)
        do_set_cpus_allowed(p, new_mask)
        if (cpumask_test_cpu(task_cpu(p), new_mask))
            goto out
        dest_cpu = cpumask_any_and(cpu_active_mask, new_mask)
        if (p->on_rq)
            arg = { p, dest_cpu }
            stop_one_cpu(cpu_of(rq), migration_cpu_stop, &arg)

    migration_cpu_stop(*data)
        ...

다양한 경우에 `set_cpus_allowed_ptr()` 을 이용하여 `p->cpus_allowd` 마스크를
설정할 수 있는데 새롭게 설정된 마스크가 현재 cpu 를 허용하지 않는다면 태스크를
migraion 하는데 이때 `check_preempt_curr()` 를 호출한다.

>NOTE  
>`migration_cpu_stop(*data)` 함수는 2.4.1. Execute Binary 참고  

#### 2.4.4. Load Balancing [section-load-balance]

    load_balance(this_cpu, *this_rq, *sd, idle, *continue_balancing)
        env = sd, this_cpu, this_rq, sched_group_cpus(sd->groups), idle,
              sched_nr_migrate_break, cpus, all
        ...
        group = find_busiest_group(&env)
        if (!group)
                goto out_balanced
        busiest = find_busiest_queue(&env, group);
        if (!busiest)
            goto out_balanced
        ...
        if (busiest->nr_running > 1)
            ...
            env.src_cpu = busiest->cpu
            env.src_rq = busiest
            ...
            cur_ld_moved = move_tasks(&env)
                tasks = &env->src_rq->cfs_tasks
                while (!list_empty(tasks))
                    ...
                    move_task(p, env)
                        deactivate_task(env->src_rq, p, 0)
                        set_task_cpu(p, env->dst_cpu)
                        activate_task(env->dst_rq, p, 0)
                        check_preempt_curr(env->dst_rq, p, 0)
            ...
            if (cur_ld_moved && env.dst_cpu != smp_processor_id())
                resched_cpu(env.dst_cpu)
                    resched_task(cpu_curr(cpu))

로드밸런싱의 일환으로 실행되어 `resched_task()` 나 `check_preempt_curr()` 를
호출한다.

### 2.5. Priority or Scheduler Class Change Activity [section-change-prior-sched]
#### 2.5.1. Setting Nice Value [section-set-nice]

    set_user_nice(*p, nice)
        ...
        on_rq = p->on_rq
        if (on_rq)
            dequeue_task(rq, p, 0)
        p->static_prio = NICE_TO_PRIO(nice)
        set_load_weight(p)
        old_prio = p->prio
        p->prio = effective_prio(p)
        delta = p->prio - old_prio
        if (on_rq)
            enqueue_task(rq, p, 0)
            if (delta < 0 || (delta > 0 && task_running(rq, p)))
                resched_task(rq->curr)

우선순위가 변경된 경우 `resched_task()` 를 호출한다.

#### 2.3.5. Change Scheduler Classes [section-change-sched-class]

    __sched_setscheduler(*p, *attr, user)
        ...
        rq = task_rq_lock(p, &flags)
        ...
        oldprio = p->prio
        ...
        prev_class = p->sched_class
        __setscheduler(rq, p, attr)
        ...
        check_class_changed(rq, p, prev_class, oldprio)
            if (prev_class != p->sched_class)
                ...
                p->sched_class->switched_to(rq, p)
                    *se = &p->se
                    if (!se->on_rq)
                        return
                    if (rq->curr == p)
                        resched_task(rq->curr)
                    else
                        check_preempt_curr(rq, p, 0)
            else if (oldprio != p->prio || dl_task(p))
                p->sched_class->prio_changed(rq, p, oldprio)
                    if (!p->se.on_rq)
                        return
                    if (rq->curr == p)
                        if (p->prio > oldprio)
                            resched_task(rq->curr)
                     else
                        check_preempt_curr(rq, p, 0)

스케쥴러클래스를 변경할 때 `resched_task()` 나 `check_preempt_curr()` 를
호출한다.

#### 2.5.3. Priority Change When Locking/Unlocking RTMutex [section-rt-mutex]

    rt_mutex_setprio(p, prio)
        check_class_chaged(rq, p, prev_class, oldprio)
            ...

rt mutex 를 lock/unlock 하는 경우에 호출되며 나머지 내용은 스케쥴러 클래스가
바뀌는 경우와 내용이 같다.

>NOTE  
>`check_class_chaged()` 는 2.3.5. Change Scheduler Classes 참고  

#### 2.5.4. Normalize RT Tasks via Magic Key 'n' [section-magic-key-n]

	normalize_rt_tasks()
        do_each_thread(g, p)
            rq = __task_rq_lock(p);
            normalize_task(rq, p)
                *prev_class = p->sched_class;
                attr = SCHED_NORMAL
                old_prio = p->prio;

                on_rq = p->on_rq;
                if (on_rq)
                    dequeue_task(rq, p, 0)
                __setscheduler(rq, p, &attr)
                if (on_rq)
                    enqueue_task(rq, p, 0)
                    resched_task(rq->curr)
                check_class_changed(rq, p, prev_class, old_prio)
        while_each_thread(g, p)

Magic Key `n` 을 눌렀을 경우 호출되며 나머지 내용은 스케쥴러 클래스가 바뀌는
경우와 내용이 같다.

>NOTE  
>`check_class_chaged()` 는 2.3.5. Change Scheduler Classes 참고  

### 2.6. Bandwidth Control Activity [section-bandwidth-ctrl]

    account_cfs_rq_runtime(cfs_rq, delta_exec)
        ...
        __account_cfs_rq_runtime(cfs_rq, delta_exec)
            cfs_rq->runtime_remaining -= delta_exec;
            expire_cfs_rq_runtime(cfs_rq)
            if (likely(cfs_rq->runtime_remaining > 0))
                return
            if (!assign_cfs_rq_runtime(cfs_rq) && likely(cfs_rq->curr))
                resched_task(rq_of(cfs_rq)->curr)

    unthrottle_cfs_rq(cfs_rq)
        ...
        if (rq->curr == rq->idle && rq->cfs.nr_running)
            resched_task(rq->curr)

bandwidth 제어의 경우에 `resched_task()` 가 호출될 가능성이 있다.

### 2.7. Various Other Sites [section-other-sites]

이외에도 다양한 (하지만 언급된 것들보다 사소한) 다른 경우의 수가 있을 수 있다.

---

## 3. Checking TIF_NEED_RESCHED flag and Calling Scheduler [section-calling-scheduler]

당연히 스케쥴러를 호출하는 곳은 커널모드 (예외문맥) 이며 이전에 실행하던
모드에 따라 (돌아가야할 모드) 스케쥴러를 호출하는 경우가 다르다.

이전에 프로세스가 유저모드로 실행중일때 예외가 발생하여 커널모드로 들어온 경우
커널모드에서 유저모드로 복귀하기전 스케쥴러가 호출될 가능성이 있다.

이전에 프로세스가 커널모드로 실행중일때 예외가 발생하여 커널모드로 들어온 경우
이전 커널제어흐름으로 복귀하기전 과 커널모드로 실행중에 preemption 이
활성화되는 시점에서 스케쥴러가 호출될 가능성이 있다.

### 3.1. Process and Processor Modes [section-proc-modes]
#### 3.1.1. User Mode and Exceptions [section-usr-mode-exception]
##### 유저프로세스(프로세서모드 : usr) 실행중 system call 처리흐름.

    - User Mode -  ---------------------+<--+
                    1.                  |   |
                                        |   |
    user mode stack     registers       |   |
                                        |   |
    +-------+           +-------+       |   |
    |   .   |           |   .   | 2.    |   |
    |   .   |           |   .   |---+   |   |
    +-------+ <- sp     |   .   |   |   |   |
    |   .   |           +-------+   |   |   |
                                    |   |   |
                                    |   |   |
    - SVC Mode - <------------------|---+   |
                 -------------------|-------+
                  4.                |
    kernel mode stack               |
                                    |
    +-------+                       |
    |   .   | <---------------------+
    |   .   | pt_regs
    |   .   |
    +-------+ <- sp
    |   .   |
    |   .   |
        .
        3.

1. swi exception occurs.  
2. save user mode register to kernel stack.  
3. handles system call.  
4. return to user from svc.  

##### 유저프로세스(프로세서모드 : usr) 실행중 irq/abt/und 처리흐름.  

    - User Mode - ----------------------+<--+
                   1.                   |   |
    user mode stack     registers       |   |
                                        |   |
    +-------+           +-------+       |   |
    |   .   |           |   .   |       |   |
    |   .   |           |   .   |---+   |   |
    +-------+ <- sp     |   .   |   |   |   |
    |   .   |           +-------+   |   |   |
                                    |   |   |
                                    |   |   |
    - IRQ/ABT/UND Mode - <----------|---+   |
                         -----------|---+   |
    irq/abt/und stack     3.        |   |   |
                                    |   |   |
    +-------+                       |   |   |
    |  spsr |   2.                  |   |   |
    +-------+ <---------------------+   |   |
    |  lr   | --------------+       |   |   |
    +-------+  4.           |       |   |   |
    |  r0   |               |       |   |   |
    +-------+ <- sp         |       |   |   |
                            |       |   |   |
    - SVC Mode - <----------|-------|---+   |
                 -----------|-------|-------+
                  6.        |       |
    kernel mode stack       |       |
                            |       |
    +-------+               |       |
    |   .   | <-------------+       |
    |   .   | <---------------------+ 4.
    |   .   | pt_regs
    |   .   |
    +-------+ <- sp
    |   .   |
        .
        5.

1. irq/abt/und exception occurs.  
2. save some user mode registers to each mode's stack.  
3. change to svc mode immediately.  
4. save all registers (user and exception mode) to kernel stack.  
5. handles irq/abt/und.  
6. return to usr from svc.  

irq/abt/und 처리에서는 해당 exception 으로 프로세서모드가 바뀌고 임시로
몇가지 레지스터만을 해당 스택에 저장한다. 그리고 나서 즉시 svc 모드로 프로세서
모드를 변경하고 유저모드 문맥복구에 필요한 모든 레지스터들을 커널모드 스택
(current 프로세스) 에 저장한후 실제 예외핸들링을 진행한다. 예외처리가 끝날때
svc 모드에서 바로 user 모드로 커널모드 스택에 저장된 문맥을 사용하여
유저프로세스로 실행흐름을 복구한다.

#### 3.1.2. Kernel Mode and Exceptions [section-kernel-mode-exception]
##### 유저프로세스를 커널모드로 혹은 커널 스레드 (프로세서모드 : svc) 실행중 irq/abt/und 처리흐름.

    - SVC Mode - ---------------------------------------+<--+
                  1.                                    |   |
    kernel mode stack                   registers       |   |
                                                        |   |
    +-------+                           +-------+       |   |
    |   .   |                           |   .   |       |   |
    |   .   | (pt_regs of user mode)    |   .   |---+   |   |
    |   .   | <- sp                     |   .   |   |   |   |
    +-------+                           +-------+   |   |   |
    |   .   |                                       |   |   |
                                                    |   |   |
    - IRQ/ABT/UND Mode - <--------------------------|---+   |
                         ---------------------------|---+   |
    irq/abt/und stack     3.                        |   |   |
                                                    |   |   |
    +-------+                                       |   |   |
    |  spsr |   2.                                  |   |   |
    +-------+ <-------------------------------------+   |   |
    |  lr   | --------------------------+           |   |   |
    +-------+   4.                      |           |   |   |
    |  r0   |                           |           |   |   |
    +-------+ <- sp                     |           |   |   |
                                        |           |   |   |
    - SVC Mode - <----------------------|-----------|---+   |
                 -----------------------|-----------|-------+
                  6.                    |           |
    kernel mode stack                   |           |
                                        |           |
    +-------+                           |           |
    |   .   |                           |           |
    |   .   | (pt_regs of user mode)    |           |
    |   .   |                           |           |
    +-------+                           |           |
    |   .   | <-------------------------+           |
    |   .   | <-------------------------------------+ 4.
    |   .   | pt_regs of kernel mode
    |   .   |
    +-------+ <- sp
    |   .   |
        .
        5.

0. (maybe in the middle of systam call handling for user process or
    kernel thread.)
1. irq/abt/und exception occurs.
2. save some kernel mode registers to each mode's stack.
3. change to svc mode immediately.
4. save all registers to kernel stack.
5. handles irq/abt/und.
6. return to svc from svc.

irq/abt/und 처리에서는 해당 exception 으로 프로세서모드가 바뀌고 임시로
몇가지 레지스터만을 해당 스택에 저장한다. 그리고 나서 즉시 svc 모드로 프로세서
모드를 변경하고 커널모드 문맥복구에 필요한 모든 레지스터들을 스택
(current 프로세스) 에 저장한후 실제 예외핸들링을 진행한다. 예외처리가 끝날때
커널모드 스택에 저장된 문맥을 사용하여 커널제어흐름을 복구한다.

>NOTE  
>there is no pt_regs of user mode for kernel thread.  

### 3.2. Return to User Mode [section-return-user-mode]
#### 3.2.1. Return from System Call [section-return-user-syscall]

    asmlinkage int
    do_work_pending(struct pt_regs *regs, unsigned int thread_flags,
                    int syscall)
    {
        do {
            if (likely(thread_flags & _TIF_NEED_RESCHED)) {
                schedule();
            } else {
                if (unlikely(!user_mode(regs)))
                    return 0;
                local_irq_enable();
                if (thread_flags & _TIF_SIGPENDING) {
                    int restart = do_signal(regs, syscall);
                    if (unlikely(restart)) {
                        return restart;
                    }
                    syscall = 0;
                } else if (thread_flags & _TIF_UPROBE) {
                    clear_thread_flag(TIF_UPROBE);
                    uprobe_notify_resume(regs);
                } else {
                    clear_thread_flag(TIF_NOTIFY_RESUME);
                    tracehook_notify_resume(regs);
                }
            }
            local_irq_disable();
            thread_flags = current_thread_info()->flags;
        } while (thread_flags & _TIF_WORK_MASK);
        return 0;
    }

시스템콜 처리의 끝부분에서 `_TIF_WORK_MASK` 를 보고 `do_work_pending()` 을
실행하는데 이때 `thread_info->flags` 에서 `_TIF_NEED_RESCHED` 을 확인하여
스케쥴러를 호출한다.

    arch/arm/include/asm/thread_info.h:

    #define _TIF_WORK_MASK    (_TIF_NEED_RESCHED | _TIF_SIGPENDING | \
                               _TIF_NOTIFY_RESUME)

`_TIF_WORK_MASK` 의 정의는 위와 같다. 어디선가 `_TIF_NEED_RESCHED` 를 설정하면
`_TIF_WORK_MASK` 에 영향을 미치게된다.

                            vector_swi()
                                |
                                v
                            local_restart()
                                |
                                v
                    ti->flags & _TIF_SYSCALL_WORK
                                |
                                v
        +-----------------------+---------------------------+
        | 0                                                 | 1
        |                                                   v
        |                                               __sys_trace()
        |                                                   |
        v                                                   v
    sys_whatever()                                      sys_whatever()
        |                                                   |
        v                                                   v
    ret_fast_syscall                                    __sys_trace_return
        |                                                   |
        v                                                   v
    tsk->flags & _TIF_WORK_MASK                         ret_slow_syscall
        |                                                   |
        |                                                   v
        |                                           ti->flags & _TIF_WORK_MASK
        |                                                   |
        v                                                   v
        +-------------------+                   +-----------+
        | 0                 | 1                 | 1         | 0
        |                   v                   |           |
    fast = 1,       fast_work_pending()         |       fast = 0
    offset = S_OFF          |                   |       offset = 0
        |                   +-------+-----------+           |
        |                           |                       |
        |                           v                       |
        |                       work_pending()              |
        |                           |                       |
        |                           v                       |
        |               do_work_pending() <== (schedule() ca|n be called here.)
        |                       |                           |
        |                       v                           |
        |                   restart == 1                    |
        |                       |                           |
        |                       v                           |
        |           +-----------+-----------+               |
        |           | 0                     | 1             |
        |           v                       v               |   ^
        |   no_work_pending()       sys_restart_syscall()   |   .
        |           |                       |               |   .
        |           |                       v               |   .
        |           |               local_restart() ........|....
        |           |                                       |
        |           v                                       |
        +-----------+---------------------------------------+
                    |
                    v
            restore_user_regs()

#### 3.2.2. Return from IRQ [section-return-user-irq]

    __vector_irq()
        __irq_usr()
            usr_entry()
            irq_handler()
            ret_to_user_from_irq()
    		    work_pending()

유저모드로 프로세서가 실행중일때 irq 가 발생한 경우 irq 를 처리한후에
`ret_to_user_from_irq()` 를 호출하여 `_TIF_WORK_MASK` 가 설정된 경우
`work_pending()` 을 호출한다. (나머지는 swi 흐름 참고)

#### 3.2.3. Return from Other Exceptions [section-return-user-other-excpnt]

    vector_dabt()
        __dabt_usr()
            usr_entry()
            dabt_helper()
            ret_from_exception()
                ret_to_user()
                    work_pending()

    vector_pabt()
        __pabt_usr()
            usr_entry()
            pabt_helper()
            ret_from_exception()
                ret_to_user()
                    work_pending()

    vector_und()
        __und_usr()
            usr_entry()
            __und_usr_fault_32()
                und_fault()
                    do_undefinstr()
                    ret_from_exception()
                        ret_to_user()
                            work_pending()

유저모드로 프로세서가 실행중일때 abt/und 예외가 발생한 경우 예외를 처리한후에
`ret_from_exception()` 를 호출하여 유저모드로 복귀를 시도한다. 이 함수는 즉시
`ret_to_user()` 를 호출하여 `_TIF_WORK_MASK` 가 설정된 경우 `work_pending()`
을 호출한다. (나머지는 swi 흐름 참고)

### 3.3. Return to Kernel Mode [section-return-kernel-mode]
#### 3.3.1. Return from IRQ [section-return-kernel-irq]

#### Supervisor Mode return from IRQ

    vector_irq()
        __irq_svc()
            svc_entry()
            irq_handler()
            if ti->preempt_count == 0 && ti->flags & _TIF_NEED_RESCHED
                svc_preempt()
            svc_exit()

`thread_info->preempt_count` 가 0 이고 `thread_info->flags` 에
`_TIF_NEED_RESCHED` 가 설정된 경우 바로 이전 커널흐름으로 복귀하지 않고
`svc_preempt()` 를 호출하여 프로세스를 스케쥴링한다.

    arch/arm/kernel/entry-armv.S:

        .align  5
    __irq_svc:
        svc_entry
        irq_handler

    #ifdef CONFIG_PREEMPT
        get_thread_info tsk
        ldr     r8, [tsk, #TI_PREEMPT]          @ get preempt count
        ldr     r0, [tsk, #TI_FLAGS]            @ get flags
        teq     r8, #0                          @ if preempt count != 0
        movne   r0, #0                          @ force flags to 0
        tst     r0, #_TIF_NEED_RESCHED
    blne    svc_preempt
    #endif

        svc_exit r5, irq = 1                    @ return from exception
     UNWIND(.fnend          )
    ENDPROC(__irq_svc)

        .ltorg

`thread_info->preempt_count == 0 && thread_info->flags & _TIF_NEED_RESCHED`
조건을 만족하는 경우에만 `svc_preempt()` 를 호출한다.

    #ifdef CONFIG_PREEMPT
    svc_preempt:
        mov     r8, lr
    1:	bl      preempt_schedule_irq            @ irq en/disable is done inside
        ldr     r0, [tsk, #TI_FLAGS]            @ get new tasks TI_FLAGS
        tst     r0, #_TIF_NEED_RESCHED
        moveq   pc, r8                          @ go again
        b       1b
    #endif

`preempt_schedule_irq()` 를 `ti->flags` 에서 `_TIF_NEED_RESCHED` 가 설정되있는
동안 계속 반복호출한다.

    asmlinkage void __sched preempt_schedule_irq(void)
    {
        enum ctx_state prev_state;

        BUG_ON(preempt_count() || !irqs_disabled());

        prev_state = exception_enter();

        do {
            __preempt_count_add(PREEMPT_ACTIVE);
            local_irq_enable();
            __schedule();
            local_irq_disable();
            __preempt_count_sub(PREEMPT_ACTIVE);

            barrier();
        } while (need_resched());

        exception_exit(prev_state);
    }

irq 처리에서 이전 문맥으로 복귀전에 스케쥴러를 호출하려고 하는 경우이며 따라서
irq 가 금지된 상태이므로 `__schedule()` 호출 전/후로 irq 를 활성/비화성화한다.
또한 `thread_info->preempt_count` 에 `PREEMPT_ACTIVE` 를 더하기/빼기한다.
`PREEMPT_ACTIVE` 를 설정후 스케쥴러를 호출하면 **선점되는 프로세스**에 대하여
`deactivate_task()` 를 호출하지 않는다. 이는 wait 시의 race 방지를 위한 것으로
`current` 의 `task->state` 를 `TASK_RUNNING` 이 아닌 다른 상태로 (예를 들면
`TASK_[INTERRUPTIBLE/UNINTERRUPTIBLE]) 로 설정한후, waitqueue 에 삽입전에
선점되었을때 해당 프로세스가 `put_prev_task()` 에서 다시 runqueue 에 삽입되도록
하여 다시 스케쥴링됨을 보장한다.

#### 3.3.2. Kernel Mode Preemption [section-kernel-preemptiom]

    include/asm-generic/preempt.h:

    #define __preempt_schedule() preempt_schedule()


    kernel/sched/core.c:

    asmlinkage void __sched notrace preempt_schedule(void)
    {
        if (likely(!preemptible()))
            return;

        do {
            __preempt_count_add(PREEMPT_ACTIVE);
            __schedule();
            __preempt_count_sub(PREEMPT_ACTIVE);
            barrier();
        } while (need_resched());
    }

`__preempt_schedule()` 를 호출하면 현재 커널모드에서 실행중인 프로세스를 선점
할 수 있다. 물론 `ti->preempt_count == 0` 이어야한다.  
`__preempt_schedule()` 는 `preempt_enable()` 이나 `preempt_check_resched()`
에서 호출된다.  
`preempt_enable()` 는 `ti->preempt_count` 를 1 감소시키고 0 인지 확인하여
`__preempt_schedule()` 를 호출하며 `preempt_check_resched()` 는
`ti->preempt_count` 가 0 인지 확인후 `__preempt_schedule()` 를 호출한다.  

즉 `ti->preempt_count` 가 0 이 되는 지점에서 `preempt_check_resched()` 가
호출되는 모든 경우에 현재 프로세스를 선점하기 위해 스케쥴러가 호출될 수 있다.

