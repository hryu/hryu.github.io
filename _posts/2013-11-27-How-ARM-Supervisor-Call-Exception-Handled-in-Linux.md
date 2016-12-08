---
layout: post
title: How ARM Supervisor Call Exception Handled in Linux
---

## Contents

1. [Introduction][section-introduction]  
2. [SVC Exception Entry and Exit] [section-svc-exception-entry]  
2.1 [SVC Entry][section-svc-entry]  
2.2 [SVC Exit][section-svc-exit]  
3. [Allocating Stack and Saving Usr Mode Context][section-allocate-stack]  
4. [Prepare to Call System Call][section-prepare-call-syscall]  
5. [Jumping to System Call Handler][section-jump-to-syscall]  
6. [Fast and Slow System Call][section-fast-slow-syscall]  
6.1. [Function Flow][section-function-call]  
6.2. [`fast_work_pending()` and `__sys_trace_return()`][section-fast-trace]  
7. [Restore User Mode Context][section-restore-user-context]  
7.1. [restore_user_regs, fast=1, offset=S_OFF(8)][section-fast-1-offset-8]  
7.2. [restore_user_regs, fast=0, offset=0][section-fast-0-offset-0]  
7.3. [Return from SVC Exception][section-return-svc]  

---

## 1. Introduction [section-introduction]  

vector_swi() 처리를 따라가면서 리눅스 시스템 콜 처리의 시작 및 근간이 되는
ARM Supervisor Exception Handling 에 대해 알아본다.

>NOTE  
>ARMv7-AR Reference Manual, B1.9.4 Supervisor call exception
> arch/arm/kernel/entry-*.S
> CPU_V7M 및 THUMB2_KERNEL, CONFIG_OABI_COMPAT, __sys_trace, arm_syscall 제외를
  포함한 코드의 일부 축약 및 생략이 있음.
> Hypervisor, Monitor Mode 언급 제외

---

## 2. SVC Exception Entry and Exit [section-svc-exception-entry]
### 2.1 SVC Entry [section-svc-entry]

    R14_svc = address of next instruction after the SWI instruction  
    SPSR_svc = CPSR
    CPSR.M[4:0] = 0b10011
    CPSR.T =0
    CPSR.I =1	(irq disabled)
    CPSR.E = CP15_reg1_EEbit
    if high vectors configured then
        PC = 0xFFFF0008
    else
        PC = 0x00000008

Usr 모드에서 swi 명령어를 실행하였을 때 ARM H/W 레벨에서 이루어지는 일을
나타낸다.
lr_svc, spsr_svc 주목.

### 2.2 SVC Exit [section-svc-exit]

    MOVS PC,R14

S 접미사를 사용하여 pc 값을 할당하면 자동으로 spsr 이 cpsr 로 복구며 모드가
전환된다.

---

## 3. Allocating Stack and Saving Usr Mode Context [section-allocate-stack]  

    arch/arm/kernel/entry-common.S:

            .align  5
    ENTRY(vector_swi)
            sub     sp, sp, #S_FRAME_SIZE

    |  .  |
    |  .  |
    +-----+ <----- sp
    |  .  |         |
    |  .  |         |
    |  .  |         |
    |  .  |         |
    |  .  |        pt_regs (18 * sizeof(long))
    |  .  |         |
    |  .  |         |
    |  .  |         |
    |  .  |         v
    +-----+ <----- sp
    |  .  |
    |  .  |

            stmia   sp, {r0 - r12}              @ Calling r0 - r12
     ARM(   add     r8, sp, #S_PC           )
     ARM(   stmdb   r8, {sp, lr}^           )   @ Calling sp, lr
            mrs     r8, spsr                    @ called from non-FIQ mode,
                                                @ so ok.
            str     lr, [sp, #S_PC]             @ Save calling PC
            str     r8, [sp, #S_PSR]            @ Save CPSR
            str     r0, [sp, #S_OLD_R0]         @ Save OLD_R0
            ...

    |    .     |
    |    .     |
    +----------+
    | r0_orig  |
    +----------+
    | spsr_svc |
    +----------+
    |  lr_svc  |
    +----------+
    |  lr_usr  |
    +----------+
    |  sp_usr  |
    +----------+
    |   r12    |
    +----------+
    |    .     |
    |    .     |
    +----------+
    |    r0    |
    +----------+ <---- sp
    |    .     |
    |    .     |

spsr_svc    : 예외발생 이전의 usr 모드에서 사용중이던 cpsr 의 값이 spsr_svc 에
              저장되어있음.  
lr_svc      : 예외발생 이전의 usr 모드에서 사용중이던 pc의 값
              (lr_svc = CPSR.T == O ? pc - 4 : pc - 2) 이 lr_svc 에 저정되어
              있음.  swi 는 decode 단계에서 발생하므로 pc-4를 하면 swi를
              일으킨 명령 바로 다음의 주소를 얻게 된다.  

    pc		fetch
    pc - 4	decode
    pc - 8	execute

위 표는 fetch -> decode -> execute 로 이어지는 파이프라인 특성때문에 pc 값이
fetch 단계의 주소를 가리키고 있어 pc 를 읽어 들이는 실제 명령수행 단계에서는
ARM 모드의 경우 8byte, Thumb 모드에서는 4 바이트만큼 항상 앞서 있다.

r0_orig     : 시그널 처리시에 시스템콜을 다시 실행해야 하는 경우 r0의 값을 다시
              복구하기 위해 사용됨.  

---

## 4. Prepare to Call System Call [section-prepare-call-syscall]

    arch/arm/kernel/entry-common.S:

        ...
        adr     tbl, sys_call_table             @ load syscall table pointer

    local_restart:
        ldr     r10, [tsk, #TI_FLAGS]           @ check for syscall tracing
        stmdb   sp!, {r4, r5}                   @ push fifth and sixth args
        ...

    |    .     |
    |    .     |
    +----------+
    |    r0    |
    +----------+ <---- sp
    |    r5    |        |
    +----------+        |
    |    r4    |        v
    +----------+ <---- sp
    |    .     |
    |    .     |

    arch/arm/kernel/entry-header.S:

    scno    .req    r7              @ syscall number
    tbl     .req    r8              @ syscall table pointer

>NOTE  
>Procedure Call Standard for the ARM Architecture (AAPCS) 참고

------

## 5. Jumping to System Call Handler [section-jump-to-syscall]

    arch/arm/kernel/entry-common.S:

    ...
    cmp     scno, #NR_syscalls              @ check upper syscall limit
    adr     lr, BSYM(ret_fast_syscall)      @ return address
    ldrcc   pc, [tbl, scno, lsl #2]         @ call sys_* routine
    ...

특정 시스템콜로 점프하여 시스템콜요청을 처리한다.  
ret_fast_syscall을 lr에 넣어둠으로써 시스템콜 복귀시 해당 심볼로 점프하도록
설정.  
ldr에 cc접미사를 사용하여 scno가 유효한 값인 경우에만 pc 를 설정.
(아니면 아래로 계속 진행)

---

## 6. Fast and Slow System Call [section-fast-slow-syscall]

fast 와 slow 의 구분은 단순히  
 a. 시스템콜 함수 호출 (sys_*()) 전에 `__sys_trace()` 처리나  
 b. 해당 태스크에 `_TIF_WORK_MASK` 설정으로 인한 시스템콜 함수 리턴 이후에
    `do_work_pending()` 처리를 하는지  
여부에 따라 결정되는데 (둘 중 하나라도 처리하면 slow)  
slow 는 다른 함수를 처리 (인수 전달 및 분기)로 인해 유저모드로의 복귀시에  
 a. 시스템콜 리턴값인 r0를 스택에 넣었다가 스택에서 다시 복구해야 하며  
 b. 시스템콜을 위한 5,6 번째 인수들이 이미 스택에서 제거 되었기 때문에 스택을
    덜 말아올려도 되는 (`S_OFF` (8 bytes))  
차이가 있다.  
fast 는 시스템콜 호출 전/후로 다른 함수처리를 하지 않았으므로 유저모드 복귀시  
 a. r0 가 corruption 되지 않아 스택으로 부터 복구할 필요없이 그냥 두면 되고  
 b. r4,r5 가 스택프레임에 포함되어있으므로 추가로 `S_OFF` 만큼 더 스택을
    말아올려야함.  

### 6.1. Function Flow [section-function-call]

    swi
     |
     v
    tsk->flags & _TIF_SYSCALL_WORK
     |
     |----------------------------------------------+
     | (0)                                          | (1)
     |                                              v
     |                                             __sys_trace
     |                                              |
     +--------------> sys_whatever() <--------------+
     |                                              |
     v                                              v
    ret_fast_syscall                               __sys_trace_return
     |                                              |
     v                                              v
    tsk->flags & _TIF_WORK_MASK                    ret_slow_syscall
     |                                              |
     +--------------------------+                   v
     | (0)                      | (1)              tsk->flags & _TIF_WORK_MASK
     v                          v                   |
    fast = 1, offset = S_OFF   fast_work_pending    +---------------+
    offset = S_OFF              |                   | (1)           | (0)
     |                          v                   |              fast = 0,
     |                         work_pending <-------+              offset = 0
     |                          |                                   |
     |                          v                                   |
     |                         do_work_pending                      |
     |                                                              |
     +------------------------------+-------------------------------+
                                    |
                                    v
                            restore_user_regs

### 6.2. `fast_work_pending()` and `__sys_trace_return()` [section-fast-trace]

    |    .     |
    |    .     |
    +----------+
    |    r0    | <=== r0 (return value of sys_whatever())
    +----------+ <---- sp
    |    r5    |        ^
    +----------+        |
    |    r4    |        |
    +----------+ <---- sp
    |    .     |
    |    .     |

이 심볼들은 각각 `do_work_pending()`을 호출하기전, `sys_whatever()` 를 호출하기
전 lr에 주소를 저장해둠으로써 함수리턴시 실행되도록 준비된다. 이후 함수 호출을
대비(인수전달 및 지역변수을 위한 레지스터 및 스택 사용) 하여 스택에 시스템콜
로부터 리턴된 r0 값을 저장하고 시스템콜의 5,6 번째 인수를 위해 사용한 스택공간
(r4,5) 을 제거한다.  

---

## 7. Restore User Mode Context [section-restore-user-context]

    arch/arm/kernel/entry-header.S:

        .macro  restore_user_regs, fast = 0, offset = 0
        ldr     r1, [sp, #\offset + S_PSR]  @ get calling cpsr
        ldr     lr, [sp, #\offset + S_PC]!  @ get pc

lr_svc를 스택에 저장된 컨택스트로부터 복구하여 pc 를 대체할 준비

        msr     spsr_cxsf, r1               @ save in spsr_svc

spsr_svc를 스택에 저장된 컨택스트로부터 복구하여 cpsr 을 대체할 준비

    #if defined(CONFIG_CPU_V6)
        strex   r1, r2, [sp]                @ clear the exclusive monitor
    #elif defined(CONFIG_CPU_32v6K)
        clrex                               @ clear the exclusive monitor
    #endif
        .if     \fast
        ldmdb   sp, {r1 - lr}^              @ get calling r1 - lr
        .else
        ldmdb   sp, {r0 - lr}^              @ get calling r0 - lr
        .endif
        mov     r0, r0                      @ ARMv5T and earlier require a nop
											@ after ldm {}^

스택에 저장된 컨택스트로부터 sp_usr, lr_usr을 포함한 (r0), r1-r12 복구한다.
복수레지스터 전송에서 ^ 이 있으면 해당 모드의 레지스터가 아니라 유저모드의
레지스터를 대상한다는 의미이다. (그래서 sp_svc, lr_svc가 아니라 sp_usr, lr_usr
에 값이 저장된다)  
만일 레지스터 목록에 pc가 명시되어있다면 위와는 다른 의미이며 spsr이 cpsr 로
암묵적으로 복사된다는 의미이며 예외복귀에서 사용된다.  

> NOTE  
RealView Compilation Tools 버전 4.0: 4.2.8 LDM 및 STM 참고  

        add     sp, sp, #S_FRAME_SIZE - S_PC

스택을 완전히 말아 올려 사용한 공간을 반환한다.

        movs    pc, lr                       @ return & move spsr_svc into cpsr

spsr 을 자동으로 cpsr로 복구하면서 svc 모드에서 usr 모드로 복귀한다.

### 7.1. restore_user_regs, fast=1, offset=S_OFF(8) [section-fast-1-offset-8]

    |    .     |
    |    .     |
    +----------+ <---- sp
    | r0_orig  |        ^
    +----------+        |
    | spsr_usr |        |    -> spsr_svc
    +----------+        |
    |  lr_svc  | <---- sp   -> lr_svc
    +----------+        ^
    |  lr_usr  |        |    -> lr_usr
    +----------+        |
    |  sp_usr  |        |    -> sp_usr
    +----------+        |
    |   r12    |        |    -> r12
    +----------+        |
    |    .     |        |        .
    |    .     |        |        .
    +----------+        |
    |    r1    |        |    -> r1
    +----------+        |
    |    r0    |        |
    +----------+        |
    |    r5    |        |
    +----------+        |
    |    r4    |        |
    +----------+ <---- sp
    |    .     |
    |    .     |

r0 는 sys_whatever() 의 리턴 이후 수정되지 않았으므로 스택으로 부터 복구하지 않아도 된다.

### 7.2. restore_user_regs, fast=0, offset=0 [section-fast-0-offset-0]

    |    .     |
    |    .     |
    +----------+ <---- sp
    | r0_orig  |        ^
    +----------+        |
    | spsr_usr |        |   -> spsr_svc
    +----------+        |
    |  lr_svc  | <---- sp   -> lr_svc
    +----------+        ^
    |  lr_usr  |        |   -> lr_usr
    +----------+        |
    |  sp_usr  |        |   -> sp_usr
    +----------+        |
    |   r12    |        |   -> r12
    +----------+        |
    |    .     |        |       .
    |    .     |        |       .
    +----------+        |
    |    r0    |        |   -> r0
    +----------+ <---- sp
    |    .     |
    |    .     |

r0 도 스택에서 복구함.

### 7.3. Return from SVC Exception [section-return-svc]

    movs	pc, lr

spsr 을 자동으로 cpsr로 복구하면서 svc 모드에서 usr 모드로 복귀.

---
