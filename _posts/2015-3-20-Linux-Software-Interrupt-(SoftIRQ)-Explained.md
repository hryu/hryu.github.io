---
layout: post
title: Linux Software Interrupt (SoftIRQ) Explained
---

## Contents
1. [Introduction][section-intro]  
2. [Opening SoftIRQ][section-open]  
3. [Raising SoftIRQ][section-raise]  
3.1. [IRQ CPU Stats Structure][section-irq-stat]  
3.2. [Raising SoftIRQ APIs][section-raise-apis]  
4. [Handling Softirq][section-handle-softirq]  
4.1. [At the End of IRQ][section-end-irq]  
4.2. [Wake up ksoftirqd][section-wakeup-softirqd]  
4.3. [__do_softirq()][section-do-softirq]  
5. [Disable/Enable Bottem Half][section-disable-bottom-hf]  
5.1. [Preemption Mask][section-preempt-mask]  
5.2. [Disable/Enable BH Explicitly][section-dis/enable-explicitly]  
5.3. [Disable/Enable BH In __do_softirq()][section-dis/enable-in-softirq]  
6. [Races on SMP in SoftIRQ Handlers][section-races-softirq-handler]  

---

## 1. Introduction [section-intro]

Linux SoftIRQ 구현과 어떻게/어떤 상황에서 softirq 가 처리되는지 알아본다.

>NOTE  
>3.15.0-rc5, Shuffling Zombie Juror  
>ARM port dependent.  

---

## 2. Opening SoftIRQ [section-open]

    *** include/linux/interrupt.h:

    enum
    {
        HI_SOFTIRQ=0,
        TIMER_SOFTIRQ,
        NET_TX_SOFTIRQ,
        NET_RX_SOFTIRQ,
        BLOCK_SOFTIRQ,
        BLOCK_IOPOLL_SOFTIRQ,
        TASKLET_SOFTIRQ,
        SCHED_SOFTIRQ,
        HRTIMER_SOFTIRQ,
        RCU_SOFTIRQ,    /* Preferable RCU should always be the last softirq */

        NR_SOFTIRQS
    };

    struct softirq_action
    {
        void    (*action)(struct softirq_action *);
    };


    *** kernel/softirq.c:

    static struct softirq_action softirq_vec[NR_SOFTIRQS]
                                                __cacheline_aligned_in_smp;

    void open_softirq(int nr, void (*action)(struct softirq_action *))
    {
        softirq_vec[nr].action = action;
    }

정적 매핑된 인덱스에 따라 정해진 핸들러들이 준비되어있으며 커널 초기화시 한번
핸들러가 할당되고 변하지 않는다.
`open_softirq()` 함수로 검색하면 각 enumeration 에 따라 할당되는 핸들러들을
찾아볼 수 있다.

---

## 3. Raising SoftIRQ [section-raise]
### 3.1. IRQ CPU Stats Structure [section-irq-stat]

    *** arch/arm/include/asm/hardirq.h:

    typedef struct {
        unsigned int __softirq_pending;
    #ifdef CONFIG_SMP
        unsigned int ipi_irqs[NR_IPI];
    #endif
    } ____cacheline_aligned irq_cpustat_t;

    *** include/linux/irq_cpustat.h:

    #ifndef __ARCH_IRQ_STAT
    extern irq_cpustat_t irq_stat[];            /* defined in asm/hardirq.h */
    #define __IRQ_STAT(cpu, member) (irq_stat[cpu].member)
    #endif

    #define local_softirq_pending() \
            __IRQ_STAT(smp_processor_id(), __softirq_pending)

    *** kernel/softirq.c

    #ifndef __ARCH_IRQ_STAT
    irq_cpustat_t irq_stat[NR_CPUS] ____cacheline_aligned;
    EXPORT_SYMBOL(irq_stat);
    #endif

각 cpu 마다 `irq_stat` 이 마련되어 있으며 `__softirq_pending` 에 위에서 살펴본
softirq 타입이 쉬프트되어 설정/해제된다. `__softirq_pending` 이 cpu 마다
마련되어 있으므로 softirq 과 관련된 처리는 해당 cpu 에만 국한되며 다른 cpu 와는
독립적으로 처리된다.

    *** include/linux/interrupt.h:

    #ifndef __ARCH_SET_SOFTIRQ_PENDING
    #define set_softirq_pending(x) (local_softirq_pending() = (x))
    #define or_softirq_pending(x)  (local_softirq_pending() |= (x))
    #endif

`local_softirq_pending()` 로 접근되는 데이터는 해당 코드를 실행중인 cpu 만의
데어터이므로 cpu 간 상호배제는 필요없다. 그러나 쓰기 접근시 local irq 를
비활성화하는 것이 중요한데 이는 interrupt 에 의한 선점으로 인해 해당 데이터에
대한 경쟁이 발생하기때문이다.  

    or_softirq_pending(1 << nr)         __do_softirq

    temp = local_softirq_pending()
    INTERRUPT -->
                                        pending = local_softirq_pending()
                                        set_softirq_pending(0);
                                        while (ffs(pending))
                                            h->action(h)
    <-- INTERRUPT
    temp |= x

잘못된 동기화에 의해 데이터가 깨지는 것과 같은 내용이지만 예를 들어 설명하면,  
`or_softirq_pending()` 하는 쪽에서 일부비트가 설정된 상태값을 읽어들인 후,
인터럽트가 발생하여 해당 제어흐름이 선점된 상태에서 `irq_exit()` 후반에
softirq 를 처리한후 다시 선점되었던 제어흐름으로 복귀하게 되면 이전에 읽어둔
값과 or 하여 비트를 설정하게 되고 실제로는 이미 모두 처리된 softirq 들에 대해서
다시 설정하는 꼴이 된다. 자세한 사항은 나머지절 참고.

## 3.2. Raising SoftIRQ APIs [section-raise-apis]

|                   | already irq disabled      |                   |
|-------------------|---------------------------|-------------------|
|                   | __raise_softirq_irqoff    |                   |
| wake up softirqd  | raise_softirq_irqoff      | raise_softirq     |

irq 가 이미 비활성화 되었을 때 사용하는 함수 두개, irq 활성화 상태에서 호출하는
함수 한개.  
irq 가 이미 비활성화 되었을 때 사용하는 함수 두개중, wakeup_softirqd 를 호촐하
는 함수 한개, 아닌것 한개.  

`local_softirq_pending()` 은 살펴본바와 같이 local irq 가 비활성화된 상태에서
조작하여야 한다. 허나 이미 local irq 가 비활성화된 상태에서
`local_irq_[save/restore]` 를 호출하는 것은 무의미하므로 그러한 경우 사용하는
해당 함수들을 호출하지 않는 함수 `__raise_softirq_irqoff()` 과
`raise_softirq_irqoff()` 이 있다.  
`__raise_softirq_irqoff()` 는 `wakeup_softirqd()` 호출하지 않는데 irq 핸들러나
softirq 핸들러와 같이 커널 흐름상 처리후 softirq 가 처리됨을 보장하는 경우에
사용한다.

---

## 4. Handling Softirq [section-handle-softirq]
### 4.1. At the End of IRQ [section-end-irq]

    handle_IRQ()
        old_regs = set_irq_regs(regs)
        irq_enter()
            rcu_irq_enter()
        generic_handle_irq()
        irq_exit()
            preempt_count_sub(HARDIRQ_OFFSET)
            if (!in_interrupt() && local_softirq_pending())
                invoke_softirq()
    set_irq_regs(old_regs)

    handle_IPI

'in_interrupt()` 확인 구문으로 인해 softirq 처리중 irq 가 발생하여 이 코드로
들어온 경우, softirq 를 처리하지 않는다. irq 이전 중단된 softirq 처리흐름은
예외에서 돌아갈 때 복귀된다. 예외에서 복귀전 다른 프로세스로 선점될 수 있다.

### 4.2. Wake up ksoftirqd [section-wakeup-softirqd]

`wakeup_softirqd()` 는 `__do_softirq()` 에서 나머지 잔여작업위해 호출하기도
하고 `raise_softirq_irqoff()` 로 softirq 작업을 추가할때도 호출되는데
`raise_softirq_irqoff()` 서는 `in_interrupt()` 가 아닌때에만
`wakeup_softirqd()` 를 호촐한다.
irq 문맥이었을 경우 이제 곧 `irq_exit()` 에서 softirq 를 처리시도할 것이기
때문에, softirq 문맥이었을 경우 예외에서 복귀할 때 다시 재개되기 때문에 깨울
필요가 없다. 아닌 경우는 깨워야만 한다.

>force_irqthreads == true 인 경우는 논외로 하자.

## 4.3. __do_softirq() [section-do-softirq]

> 아래는 간략화 버전임.

    *** kernel/softirq.c:

    asmlinkage __visible void __do_softirq(void)
    {
        pending = local_softirq_pending();
        __local_bh_disable_ip(_RET_IP_, SOFTIRQ_OFFSET);

아직 hardirq 가 비활성화된 상태임을 숙지.
해당 cpu 의 처리해야할 softirq 비트들을 읽어들인다.
bottom half 를 비활성화하여 이후의 softirq 실행을 막는다.

    restart:
        set_softirq_pending(0);
        local_irq_enable();

softirq 비트를 0으로 초기화.
hardirq 를 다시 활성화한다. 이때부터 irq 가 방생하여 현재 문맥을 선점할 수
있다. 위에서 bh 를 비활성화하지 않았다면 다시 `__do_softirq()` 실행될 수
있으므로 문제가 될 수 있겠지.

        h = softirq_vec;
        while ((softirq_bit = ffs(pending))) {
                h += softirq_bit - 1;
                vec_nr = h - softirq_vec;
                h->action(h);
                h++;
                pending >>= softirq_bit;
        }

활성화된 비트들을 확인하며 해당 핸들러를 실행 반복한다.

        local_irq_disable();
        pending = local_softirq_pending();

해당 cpu 의 softirq pending 비트변수에 대한 race 방지를 위해 irq 를 비활성화한
후 다시 softirq pending 비트를 읽어 들인다.

        if (pending) {
                if (time_before(jiffies, end) && !need_resched() &&
                    --max_restart)
                        goto restart;
                wakeup_softirqd();
        }

softirq 핸들러를 처리하는 동안 irq 가 발생하여 pending 된 추가 softirq 가
있다면 다시 이 문맥에서 softirq 처리를 시도하는데 1. 시간, 2. resched flag,
3. max_restart 조건들을 확인하여 다시 핸들러처리를 시도한다. 조건이 만족되지
않으면 여기서 처리하지 않고 `wakeup_softirqd()` 로 softirq 데몬을 깨우는 것으로
마무리한다.

        __local_bh_enable(SOFTIRQ_OFFSET);
    }

다시 bh 를 활성화한다.

---

## 5. Disable/Enable Bottem Half [section-disable-bottom-hf]
### 5.1. Preemption Mask [section-preempt-mask]

                2019      15               7               0
               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
               |1|1|   4   |      7      |1|       8       |
               +-+-+-------+-------------+-+---------------+
                 | |       |             | |               |
                 v |       |             | |               |
    PREEMPT_ACTIVE |       |             | |               |
                   v       |             | |               |
          NMI_OFFSET       |             | |               |
                           v             | |               |
              HARDIRQ_OFFSET             | |               |
                                         v |               |
                    SOFTIRQ_DISABLE_OFFSET |               |
                                           v               |
                              SOFTIRQ_OFFSET               |
                                                           v
                                              PREEMPT_OFFSET

- `SOFTIRQ_DISABLE_OFFSET` softirq 비활성화 카운트.  
  bottom half 와 자료구조를 동기화하여야 하는 경우 각 문맥에서
  `local_bh_enable/disable()` 를 이용하여 bh 를 활성/비활성화 시킬 수 있다.
- `SOFTIRQ_OFFSET` __do_softirq() 에서 핸들러가 처리중임을 나타낸다.  
  이 플래그가 없이 `SOFTIRQ_DISABLE` 만으로는 softirq 가 처리중 (처리중 bh 를
  비활성화하기 때문에) 인지 단지 그냥 bh 가 비활성화된 것인지 구별할 수 없다.

### 5.2. Disable/Enable BH Explicitly [section-dis/enable-explicitly]

    __local_bh_disable_ip(SOFTIRQ_LOCK_OFFSET)
    local_bh_disable()
        __local_bh_disable_ip(_THIS_IP_, SOFTIRQ_DISABLE_OFFSET)

비활성화

| 함수                  | ksoftirqd 깨움    |
|-----------------------|-------------------|
| _local_bh_enable()    | X                 |
| local_bh_enable()     | O                 |

활성화

### 5.3. Disable/Enable BH In __do_softirq() [section-dis/enable-in-softirq]

    *** kernel/softirq.c:

    __do_softirq()
        ...
        __local_bh_disable_ip(_RET_IP_, SOFTIRQ_OFFSET)
        ...
        local_irq_enable()
        ...
        local_irq_disable()
        ...
        __local_bh_enable(SOFTIRQ_OFFSET)

preemption count 에 SOFTIRQ_OFFSET 을 더하여 softirq 를 비활성화하거나 빼서 softirq 을 활성화한다. 인터럽트가 활성화된 후 __do_softirq() 가 선점된 경우
`in_serving_softirq()` 를 이용하여 softirq 가 처리 중이었는지 판단할 수 있다.

    *** include/linux/preempt_mask.h:

    #define softirq_count() (preempt_count() & SOFTIRQ_MASK)
    ...

    #define in_softirq()            (softirq_count())
    #define in_serving_softirq()    (softirq_count() & SOFTIRQ_OFFSET)

`in_softirq()` 는 softirq 가 처리 중이거나 bh 가 비활성화되었을 경우를 판단할
수 있고 `in_serving_softirq()` 는 softirq 가 처리 중인 경우를 판단할 수 있다.

---

## 6. Races on SMP in SoftIRQ Handlers [section-races-softirq-handler]

살펴본 바와 같이 softirq 는 각 cpu 별로 완전히 독립적으로 raise/handling 된다.
따라서 어떠한 동일한 softirq 핸들러는 별도의 동기화를 사용하지 않는 한 서로
다른 cpu 상에서 자원을 경쟁하며 실행될 수도 있다. 이에 대한 고려가 필요하다.
