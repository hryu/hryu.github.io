---
layout: post
title: Unalignment Fault Explained on ARM Linux
---


## Contents

* [Introduction][section-introduction]
* [Alignment Checking and SCTLR Register][section-alignment-checking]
* [Allowed Unalignment accesses][section-allowed-unalignment]
* [Unalignment Access Restrictions][section-unalignment-restrictions]
* [How Aligment Fault Handled in Linux][section-how-aligment-fault-handled]

---

## Introduction [section-introduction]

ARMv7+ 아키텍처에서 어떠한 경우에 Alignment fault 가 발생하고 Linux 가 이를 어떻게 처리하고 있는지 **간략하게** 알아본다.

>NOTE  
ARMv7-AR Reference Manual 참고  
 - A3.2 Alignment support  
 - B3.12.4 Alignment faults  
 - B3.13 Exception resporting in a VMSA implementation

## Alignment Checking and SCTLR Register [section-alignment-checking]
	arch/arm/mm/proc-v7-2level.S:
	
	/*   AT                                                                 
	 *  TFR   EV X F   I D LR    S                                          
	 * .EEE ..EE PUI. .T.T 4RVI ZWRS BLDP WCAM                              
 	 * rxxx rrxx xxx0 0101 xxxx xxxx x111 xxxx < forced                     
 	 *   01    0 110       0011 1100 .111 1101 < we want                    
 	 */

A, bit[1] : Alignment check enable  
 0 : Alignment fault checking disabled.

SCTRL은 리눅스에서 위와 같이 세팅되며 A가 0이므로 몇가지 허용되는 Unalignment access 에 대해서 Alignment fault 가 발생하지 않게 된다. 물론 이외에는 fault 가 발생.

## Allowed Unalignment accesses [section-allowed-unalignment]

| instructions                                 | Alignment |
|----------------------------------------------|-----------|
| LDRH, LDRHT, LDRSH, LDRSHT, STRH, STRHT, TBH | Halfword  |
| LDR, LDRT, STR, STRT, PUSH/POP (T3,A2 Only)  | Word      |

* 언급된 명령어를 제외하고 Alignment Fault 가 발생함.

>NOTE  
SIMD 명령어 제외됨.  
도대체 T3, A2 가 뭐야.

## Unalignment Access Restrictions [section-unalignment-restrictions]
해당 access 가 fault 를 발생시키지 않는 허용되는 unaligned access 라하더라도  
* pc 레지스터를 상대로 하지 마라.  
* Device 나 Strongly-ordered 를 상대로 하지 마라.

## How Aligment Fault Handled in Linux [section-how-aligment-fault-handled]
### !CONFIG_ALIGNMENT_TRAP
이 경우 발생한 alignment fault 에 대해 어떠한 처리도 하지 않는다.

	arch/arm/mm/abort-ev7.S:
	
	ENTRY(v7_early_abort)
	...
	mrc		p15, 0, r1, c5, c0, 0		@ get FSR
	mrc		p15, 0, r0, c6, c0, 0		@ get FAR
	...
	b		do_DataAbort

각 DFSR 과 DFAR 을 읽고 `do_DataAbort()` 로 점프  

	DFSR.FS[10,3:0] (short format) : 00001
	혹은
	DFSR.STATUS[5:0] (long format) : 100001

이제 다음 코드로 넘어가는데

	arch/arm/mm/fault.c:
	
	asmlinkage void __exception
	__DataAbort(unsigned long addr, unsigned int fsr, struct pt_regs *regs)
	{
		const struct fsr_info *inf = fsr_info + fsr_fs(fsr);
		struct siginfo info;

		if (!inf->fn(addr, fsr & ~FSR_LNX_PF, regs))
			return;
		... (이후 계속)

fsr 값 (00001) 에 따라 미리 설정된 테이블 에서 함수를 호출.

	arch/arm/mm/fsr-2level.c:

	static struct fsr_info fsr_info[] = {
		...
		{ do_bad, SIGBUS,  BUS_ADRALN, "alignment exception"},
		...
	};

결국 `do_bad()` 호출하며 이 함수는 1을 리턴

		... (__DataAbort에서 이어짐)
		printk(KERN_ALERT "Unhandled fault: %s (0x%03x) at 0x%08lx\n",
				inf->name, fsr, addr);

		info.si_signo = inf->sig;
		info.si_errno = 0;
		info.si_code  = inf->code;
		info.si_addr  = (void __user *)addr;
		arm_notify_die("", regs, &info, fsr, 0);
	}

`arm_notify_die()` 를 호출하여 유저모드의 경우 SIGBUS 시그널을, 커널모드인 경우 SIGSEGV 웁스를 유발한다.

### CONFIG_ALIGNMENT_TRAP
발생한 Alignment fault (exception) 에 대하여 커널이 처리를 시도하게 된다.

#### Install Alignment Fault Handler
`CONFIG_ALIGNMENT_TRAP` 이 설정된 경우 `do_alignment()` 가 `do_bad()` 대신 예외 핸들러로 등록되며 해당 함수에서 Alignment Fault 를 처리하게 된다.

	arch/arm/include/asm/cp15.h:
	
	...
	#define CR_A    (1 << 1)        /* Alignment abort enable */
	...

	arch/arm/mm/alignment.c:
	
	static int __init alignment_init(void)
	{
		...
	#ifdef CONFIG_CPU_CP15
		if (cpu_is_v6_unaligned()) {

unaligned access 가 지원되는 cpu의 경우 SCTRL::CR_A를 0으로 설정하도록한다. v7 인 경우 true.

			cr_alignment &= ~CR_A;
			cr_no_alignment &= ~CR_A;
			set_cr(cr_alignment);
			ai_usermode = safe_usermode(ai_usermode, false);

여기서는 유저모드의 Alignment fault 에 대하여 처리할 것인지 여부를 설정한다.
`ai_usermode`는 기본값이 0이며 디폴트로 유저모드 Alignment fault 처리를 지원하지만 부팅시 `alignment=1` 를 파라미터로 넘김으로써 기능을 끌 수 있다.

		}
	#endif

		hook_fault_code(FAULT_CODE_ALIGNMENT, do_alignment, SIGBUS, BUS_ADRALN,
		...

`hook_fault_code()` 는 `fsr_info` 테이블 (arch/arm/mm/fsr-*level.c)을 수정하여 exception handler 를 바꿔치기 할 수 있는 함수임. 여기서는 `DFSR` 레지스터의 값이 `FAULT_CODE_ALIGNMENT` (1 on non-lpae) 일때 해당 예외를 `do_alignment()` 함수가 대신하며 처리 실패시 `SIGBUS` 시그널을 보내도록 설정한다.

아래 해당 함수를 참고. fsr 은 `DFSR` 이며 ifsr 은 `IFSR` 임. 아래 두 함수는 다른 경우에도 사용되며 한 예로 hw breakpoint 를 걸때도 사용된다. (arch/arm/kernel/hw_breakpoint.c::arch_hw_breakpoint_init 참고)

	arch/arm/mm/fault.c:
	
	void __init
	hook_fault_code(int nr, int (*fn)(unsigned long, unsigned int, struct pt_regs *),
					int sig, int code, const char *name)
	{
		if (nr < 0 || nr >= ARRAY_SIZE(fsr_info))
			BUG();

		fsr_info[nr].fn   = fn;
		fsr_info[nr].sig  = sig;
		fsr_info[nr].code = code;
		fsr_info[nr].name = name;
	}
	...
	
	void __init
	hook_ifault_code(int nr, int (*fn)(unsigned long, unsigned int, struct pt_regs *),
					int sig, int code, const char *name)
	{
		if (nr < 0 || nr >= ARRAY_SIZE(ifsr_info))
			BUG();

		ifsr_info[nr].fn   = fn;
		ifsr_info[nr].sig  = sig;
		ifsr_info[nr].code = code;
		ifsr_info[nr].name = name;
	}
	...

#### Handling Alignment Fault
이 부분은 어젠가 기회가 되면...
