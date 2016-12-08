---
layout: post
title: How Signal Handler Called in ARM Linux
---

## Contents

1. [Introduction][section-introduction]  
2. [Save Existing User Mode Context on Kernel Mode Stack]
    [section-save-existing-usr-context]  
3. [Modify Existing User Mode Context on Kernel Mode Stack]
    [section-modify-existing-usr-context]  
4. [Return from Signal Handler to Kernel Mode][section-return-from-signal]  
4.1 [Signal Return Page][section-signal-return-page]  
4.2 [sigreturn System Call][section-sigreturn]  
5. [Call Flow][section-call-flow]

---

## 1. Introduction [section-introduction]  

ARM Linux 를 중심으로 어떻게 커널내에서 유저모드의 시그널 핸들러가 호출되고
다시 이전의 유저 흐름으로 복귀할 수 있는지 알아본다.

>NOTE  
**참고**  
- arch/arm/kernel/signal.c  
- arch/arm/kernel/entry-common.S  
- arch/arm/kernel/sigreturn_codes.S  
- arch/arm/kernel/process.c  
realtime signal 는 흐름에 있어 대동소이하므로 언급하지 않는다.

---

## 2. Save Existing User Mode Context on Kernel Mode Stack [section-save-existing-usr-context]

    do_work_pending()
        do_signal()
            handle_signal()
                setup_frame()
    ++>             setup_sigframe()
                    setup_return()
    restore_user_regs()

            +----------+<-------+
            | r0_orig  |        |
        +---+----------+        |
        |   | spsr_svc |        |
        |   +----------+        |
        |   |  lr_svc  |        |
        |   +----------+        |
        |   |  lr_usr  |        |
    +---+   +----------+    sizeof(struct pt_regs)
    |   |   |  sp_usr  |        |
    |   |   +----------+        |
    |   |   |   r12    |        |
    |   |   +----------+        |
    |   |   |    .     |        |
    |   |   |    .     |        |
    |   |   +----------+        |
    |   |   |    r0    |        |
    |   +---+----------+ <------+
    |       |    .     |
    |       |    .     | <----- sp
    |
    |       Kernel Mode Stack
    |
    |       |    .     |
    |       |    .     |
    |       +----------+ <------+ <-----------------+-----------+
    |       | retcode1 |        |                   |           |
    |       +----------+ sizeof(unsigned long)      |           |
    |       | retcode0 |        |                   |           |
    |       +----------+ <------+                   |           |
    |       |    .     |        |                   |           |
    |   +-->|    .     |        |       sizeof(struct sigframe) |
    |   |   |    .     |        |                   |           |
    +==>+   |    .     | sizeof(struct ucontext)    |           |
        |   |    .     |        |                   |           |
        +-->|    .     |        |                   |           |
            |    .     |        |                   |           v
            +----------+ <------+-------------------+<--------- sp
            |    .     |
            |    .     |

            User Mode Stack

먼저 커널모드 스택에 이미 저장되어 있는 유저모드컨택스트(pt_regs) 를 유저모드
스택에 추가 공간을 할당한 후 복사(ucontext.uc_mcontex) 하는데 이는 기존유저모드
컨택스트의 흐름으로 복귀하는 대신에 유저모드에서 시그널핸들러를 호출하기 위한
것으로 커널모드 스택의 pt_regs 의 레지스터들을 수정해야 하기 때문이다.

---

## 3. Modify Existing User Mode Context on Kernel Mode Stack [section-modify-existing-usr-context]
### 3.1 Prepare to Call Signal Handler [section-prepare-signal-handler]

    do_work_pending()
        do_signal()
            handle_signal()
                setup_frame()
                    setup_sigframe()
    ++>             setup_return()
    restore_user_regs()

	+----------+
	| r0_orig  |
	+----------+
	| spsr_svc |    <== modified spsr_svc (clear flags, etc)
	+----------+
	|  lr_svc  |    <== address of signal handler
	+----------+
	|  lr_usr  |    <== address of return code
	+----------+
	|  sp_usr  |    <== address of signal frame on user mode stack
	+----------+
	|   r12    |
	+----------+
	|    .     |
	|    .     |
	+----------+
	|    r0    |    <== signal number
	+----------+
	|    .     |
	|    .     |

    Kernel Mode Stack

유저모드컨택스트의 원래 흐름으로 복귀하지 않고 시그널핸들러로 점프하기 위해서
lr_svc 와 spsr_svc, sp_usr 를 수정해둔다. 이대로 진행하여 restore_user_regs 이
실행되면 커널모드에서 유저모드로 복귀하면서 동시에 시그널핸들러가 설정된 스택에
서 실행된다.  
lr_usr 는 sigreturn 시스템콜를 호촐하도록 준비된 코드를 가리키게 해놓아 시그널
핸들러로부터 복귀 (`mov    pc, lr`) 하는 순간 해당 코드로 분기하여 해당
프로세스가 다시 커널모드로 들어도록 한다.

---

## 4. Return from Signal Handler to Kernel Mode [section-return-from-signal]
### 4.1 Signal Return Page [section-signal-return-page]

    |    .     |
    |    .     |
    +----------+ <----------------------------------------------+
    |    .     |                                                |
    |    .     |                                                |
    |    .     |                                                |
    |    .     |                                                |
    |    .     |                                                |
    +----------+ <------+                         1 page vma dubbed [sigpage]
    |          |        |                                       |
    |          |        +<= unsigned long sigreturn_codes[7]    |
    |          |        |                                       |
    +----------+ <------+<-- signal_return_offset (random)      |
    |    .     |                                                |
    |    .     |                                                |
    +----------+ <---- mm->context.sigpage ---------------------+
    |    .     |
    |    .     |

    Process Virtual Address Space

arch/arm/kernel/process.c::arch_setup_additional_pages() 함수는 프로세스가
`exec()` 를 실행하며 새로운 프로세스 주소 영역을 구성할 때 호출되어 별도의
페이지를 할당하고 vma를 구성, 유저모드에서 접근/실행 가능하도록 해둔다.  
해당 페이지에는 sigreturn 시스템콜을 호출하도록하는 코드가 복사되있어 준비되어
있다. 시그널 핸들러에서 복귀하면 (`mov    pc, lr`) 실행되어 sigreturn 시스템콜
을 호출, 커널모드로 다시 복귀하도록 한다.

    arch/arm/kernel/sigreturn_codes.S:

    mov     r7, #(__NR_sigreturn - __NR_SYSCALL_BASE)
    swi     #(__NR_sigreturn)|(__NR_OABI_SYSCALL_BASE)

EABI 와 OABI 양쪽에서 문제가 없도록 작성된 것을 보시라.

    arch/arm/kernel/entry-common.S:

    sys_sigreturn_wrapper:
        add     r0, sp, #S_OFF
        mov     why, #0         @ prevent syscall restart handling
        b       sys_sigreturn
    ENDPROC(sys_sigreturn_wrapper)

결국 sys_sigreturn_wrapper 를 호출한다.

### 4.2 sigreturn System Call [section-sigreturn]

    vector_swi()
        sys_sigreturn()
    ++>     restore_sigframe()
    restore_user_regs()

sys_sigreturn() 은 결국 restore_sigframe 을 호출한다. 해당함수에서 이전에 유저
모드 스택에 저장해놓은 (원래 커널모드 스택에 저장했던) 유저모드컨택스트를 다시
커널모드 스택에 복구한다.  
또하나 set_current_blocked() 를 호출, 결국 current->flags 를 다시 설정하여
경우에 따라 `_TIF_SIGPENDING` 를 해제한다.

            +----------+<-------+
            | r0_orig  |        |
        +-->+----------+        |
        |   | spsr_svc |        |
        |   +----------+        |
        |   |  lr_svc  |        |
        |   +----------+        |
        |   |  lr_usr  |        |
    +==>+   +----------+    sizeof(struct pt_regs)
    |   |   |  sp_usr  |        |
    |   |   +----------+        |
    |   |   |   r12    |        |
    |   |   +----------+        |
    |   |   |    .     |        |
    |   |   |    .     |        |
    |   |   +----------+        |
    |   |   |    r0    |        |
    |   +-->+----------+ <------+
    |       |    .     |
    |       |    .     | <----- sp
    |
    |       Kernel Mode Stack
    |
    |       |    .     |
    |       |    .     |
    |       +----------+ <------+ <-----------------+
    |       | retcode1 |        |                   |
    |       +----------+ sizeof(unsigned long)      |
    |       | retcode0 |        |                   |
    |       +----------+ <------+                   |
    |       |    .     |        |                   |
    |   +---|    .     |        |       sizeof(struct sigframe)
    |   |   |    .     |        |                   |
    +---+   |    .     | sizeof(struct ucontext)    |
        |   |    .     |        |                   |
        +---|    .     |        |                   |
            |    .     |        |                   |
            +----------+ <------+-------------------+
            |    .     |
            |    .     |

            User Mode Stack

이대로 진행하여 restore_user_regs를 실행하면 커널모드로 전환하기 이전의
유저모드실행 흐름으로 복귀된다.

---

## 5. Call Flow [section-call-flow]

    User Mode           Kernel Mode
    |                       |

    .                       .
    .                       .
    .                       .

    | normal user flow
    .
    .
    .
    | exception occurs
    +---------------------->+
                            |   exception entry (save user mode registers) ---+
                            |   handle exception                              |
                            .                                                 |
                            .                                                 |
                            .                                                 |
                            |   do_work_pending                               |
                            |       schedule                                  |
                            .                                                 |
                            .                                                 |
                            .                                                 |
                            |           or                                    |
                            |       do_signal                                 |
                            |           handle_signal                         |
                            |               setup_frame (modify svc stack) -+ |
                            |   restore_user_regs                           | |
                            |   return from exception                       | |
        +<------------------+                                               | |
        | signal handler <==================================================+ |
        .                                                                     |
        .                                                                     |
        . (exception occurrence possible)                                     |
        .                                                                     |
        .                                                                     |
        | call sigreturn                                                      |
        +------------------>+                                                 |
                            |   vector_swi (save user mode registers)         |
                            |       sys_sigreturn                             |
                            |           restore_sigframe (modify svc stack) <=+
                            |   do_work_pending
                            |       schedule
                            .
                            .
                            .
                            |           or
                            |       do_signal (other signal handling possible)
                            .
                            .
                            .
                            |   restore_user_regs
                            |   return from exception
    +<----------------------+
    | normal user flow
    .
    .
    .

>NOTE  
exception occurrence (including irq) in kernel mode omitted for simplicity.  
