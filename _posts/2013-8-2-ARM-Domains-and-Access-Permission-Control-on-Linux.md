---
layout: post
title: ARM Domains and Access Permission Control on Linux
---

# ARM Domains and Access Permission Control on Linux

---

## Contents

1. [Introduction][section-introduction]
2. [ARM Domains Configuration][section-arm-domains-configuration]
3. [How Linux Controls Domains][section-how-linux-controls-domains]
4. [ARM Access Permissions Configuration][section-arm-access-permissions-configuration]
5. [How Linux Controls Access Permissions][section-how-linux-controls-access-permissions]

---

### Introduction [section-introduction]

ARM 에서 Domain 과 AP는 어떻게 설정되며 Linux 에서 해당 설정을 어떻게
사용하는지 살펴본다.

> **NOTE**  
> 본 문서는 ARMv7 과 short descriptor format (2 Level Page Table) 만 설명한다.
> ARMv7-AR Reference Manual (Issue C), B3.7 Memory access control 참고

---

### ARM Domains Configuration [section-arm-domains-configuration]

#### First Level Descriptor

     |31                     10| 9|8      5| 4| 3| 2| 1| 0|
     | Page Table Base Address |  | Domain |        | 0| 1|

bits[1:0]

* 0x01 means Page Table
* 0x10 means Section

Domain bits[8:5]

* 2nd level descriptor에는 존재 하지 않으며 1st level page table
  descriptor의 domain 설정을 상속한다.
* section descriptor 에도 설정이 존재.
* DACR 의 index 역할을 하기 때문에 0 ~ 15 까지 총 4비트가 필요함.

#### DACR, Domain Access Control Register

Dn, bits[(2n+1):2n]

2비트씩 나누어서 각 16개의 memory domain에 access permission 들을 정의할 수 있음

* 0x00 No access  
  어떠한 접근도 Domain fault 를 일으킨다.

* 0x01 Clients  
  translation table descriptor의 AP 접근권한을 확인하여 오류시 permission fault를 일으킨다.

* 0x11 Managers  
  해당 타입에 대해서는 접근권한을 확인하지 않으므로 permission fault 가 일어나지 않는다.

---

### How Linux Controls Domains [section-how-linux-controls-domains]
#### Setting Domain Access Control Register
##### 1. Boot Time
	arch/arm/mm/Kconfig:
	
	config CPU_USE_DOMAINS
		bool
		help
		  This option enables or disables the use of domain switching
		  via the set_fs() function.

해당 설정은 ARMv7에서는 활성화 되지 않는다. ARMv6K까지 지원된다.
(CPU_32v6K가 뭐야...몰라...)

	__enable_mmu:
	...
	#ifndef CONFIG_ARM_LPAE
		mov     r5, #(domain_val(DOMAIN_USER, DOMAIN_MANAGER) | \
                      domain_val(DOMAIN_KERNEL, DOMAIN_MANAGER) | \
                      domain_val(DOMAIN_TABLE, DOMAIN_MANAGER) | \
                      domain_val(DOMAIN_IO, DOMAIN_CLIENT))
		mcr     p15, 0, r5, c3, c0, 0	@ load domain access register
		...
	#endif
		...
	ENDPROC(__enable_mmu)

시스템에 부팅될 때 기본값이 위와 같이 설정된다.
DOMAIN_* 는 다음에서 찾을 수 있다.

	arch/arm/include/asm/domain.h:
	
	#define DOMAIN_KERNEL   2
	#define DOMAIN_TABLE    2
	#define DOMAIN_USER     1
	#define DOMAIN_IO       0

domain 종류를 정의 하였다. `DOMAIN_TABLE` 은 `DOMAIN_KERNEL` 과 같이 취급됨을
주의하자 (뭐야. 쓰지도 않으면서)

	#define DOMAIN_NOACCESS 0
	#define DOMAIN_CLIENT   1
	#ifdef CONFIG_CPU_USE_DOMAINS
	#define DOMAIN_MANAGER  3
	#else
	#define DOMAIN_MANAGER  1
	#endif

	#define domain_val(dom,type)    ((type) << (2*(dom)))

domain의 접근 권한 타입을 정의하였다.
`CONFIG_CPU_USE_DOMAINS` 설정이 없으면 `DOMAIN_MANAGER` 를 `DOMAIN_CLINET`
 와 동일하게 취급하고 있음에 주의하자. 즉 `DOMAIN_KERNEL`에 대해서도 `DOMAIN_CLIENT` 타입으로 설정함으로써 Page Table Descriptor의 AP[2:0] 필드에 접근권한을 따른다는 얘기다.

이로 인해 초기 부팅시 DACR 은 다음과 같이 설정된다.

* !CONFIG_CPU_USE_DOMAINS

		|31      06|0504|0302|0100|
        | not used | 0 1| 0 1| 0 1|

|   |  DOMAIN |  TYPE  |
|---| --------|--------|
| 0 |    IO   | CLIENT |
| 1 |   USER  | CLIENT |
| 2 |  KERNEL | CLIENT |

* CONFIG_CPU_USE_DOMAINS

		|31      06|0504|0302|0100|
        | not used | 1 1| 1 1| 0 1|

|   |  DOMAIN |   TYPE  |
|---| --------|---------|
| 0 |    IO   | CLIENT  |
| 1 |   USER  | MANAGER |
| 2 |  KERNEL | MANAGER |

총 3가지 종류의 domain이 있으며 각 도메인에 대하여 타입을 할당.

| CONFIG_CPU_USE_DOMAINS | DOMAIN_KERNEL/USER |
|------------------------|--------------------|
| 1						 | DOMAIN_MANAGER	  |
| 0						 | DOMAIN_CLIENT	  |

결국 CONFIG_CPU_USE_DOMAINS에 따라 DOMAIN_MANAGER를 사용하는냐 그렇지 않냐의 차이임.
 
	arch/arm/kernel/traps.c:
	
	void __init early_trap_init(void *vectors_base)
	{
		...
		modify_domain(DOMAIN_USER, DOMAIN_CLIENT);
		...
	}

이후 (CONFIG_CPU_USE_DOMAINS == true) `paging_init()`에서 `DOMAIN_USER` 도메인의 타입을 `DOMAIN_CLIENT` 로 변경한다. (`init_task`에 영향을 줌)

* CONFIG_CPU_USE_DOMAINS

|   |  DOMAIN |   TYPE  |
|---| --------|---------|
| 0 |    IO   | CLIENT  |
| 1 |   USER  | CLIENT  |
| 2 |  KERNEL | MANAGER |

* !CONFIG_CPU_USE_DOMAINS  
변경없음

##### 2. Kernel process v.s. User process

	arch/arm/include/asm/uaccess.h:
	...
	#define KERNEL_DS       0x00000000
	...
	#define USER_DS         TASK_SIZE
	#define get_fs()        (current_thread_info()->addr_limit)
	...
	static inline void set_fs(mm_segment_t fs)
	{
		current_thread_info()->addr_limit = fs;
		modify_domain(DOMAIN_KERNEL, fs ? DOMAIN_CLIENT : DOMAIN_MANAGER);
	}

`CONFIG_CPU_USE_DOMAINS` 설정이 1이어야 의미가있다. `set_fs()` 를 통해 커널/유저 프로세스 주소 공간 한계 및 도메인 전환이 가능. 커널스레드는 `KERNEL_DS`, 유저프로세스는 `USER_DS` 사용.
인수에 따라 DOMAIN_KERNEL의 타입이 변경됨. (다른 도메인에는 영향없음)

|               | KERNEL_DS      | USER_DS       |
|---------------|----------------|---------------|
| addr_limit    | 0              | TASK_SIZE     |
| DOMAIN_KERNEL | DOMAIN_MANAGER | DOMAIN_CLIENT |

##### 2. Context Switch
	arch/arm/kernel/entry-armv.S:
	
		ENTRY(__switch_to)
		...
	#ifdef CONFIG_CPU_USE_DOMAINS
		ldr		r6, [r2, #TI_CPU_DOMAIN]
	#endif
		...
	#ifdef CONFIG_CPU_USE_DOMAINS
		mcr		p15, 0, r6, c3, c0, 0	@ Set domain register
	#endif
 		...
		ENDPROC(__switch_to)

`CONFIG_CPU_USE_DOMAINS` 활성화 시에만 적용되며 프로세스 전환시 전활할 프로세스의 `cpu_domain`을 읽어 이를 DACR 에 설정한다.

> NOTE  
> 기본적으로 커널스레드/유저스레드가 DOMAIN_KERNEL에 대해 다른 cpu_domain 값을 가지고 있으며 또한 set_fs() 로 변경되었을 수 있다.

##### 3. CONFIG_USE_CPU_DOMAINS and ARMv7
ARMv7에서는 USE_CPU_DOMAINS가 사용되지 않으므로 `DOMAIN_KERNEL`에 대해서도
`DOMAIN_CLINET` 가 설정되며 커널/유저스페이스 관계없이 Page Table의 AP[2:0] 에 따라 권한이 부여된다. 또한 `__switch_to` 나 `cpu_v7_set_pte_ext`에서 관련 코드는 동작하지 않는다.

#### Setting 1st Level Page Table Descriptor
	arch/arm/include/asm/pgalloc.h:

	...	
	#define _PAGE_USER_TABLE	(PMD_TYPE_TABLE | PMD_BIT4 | PMD_DOMAIN(DOMAIN_USER))
	#define _PAGE_KERNEL_TABLE	(PMD_TYPE_TABLE | PMD_BIT4 | PMD_DOMAIN(DOMAIN_KERNEL))
	...
	static inline void
	pmd_populate_kernel(struct mm_struct *mm, pmd_t *pmdp, pte_t *ptep)
	{
		__pmd_populate(pmdp, __pa(ptep), _PAGE_KERNEL_TABLE);
	}

	static inline void
	pmd_populate(struct mm_struct *mm, pmd_t *pmdp, pgtable_t ptep)
	{
		__pmd_populate(pmdp, page_to_phys(ptep), _PAGE_USER_TABLE);
	}
	...

각 커널/유저용 pmd 가 생성될 때, 해당 타입에 맞는 domain이 descriptor 의 domain 필드에 설정됨.

#### User Address Space Access from Kernel Mode and TUSER() Macro
* CONFIG_CPU_USE_DOMAINS  
경우에 따라 (유저 R/O 페이지에 대한 커널모드 접근권한은 R/W) 유저페이지의 접근권한이 커널모드에서와는 다르게 된다.  
`t` suffix 를 `ldr/str` 에 추가함으로써 명령이 유저모드(비권한 프로세서모드)에서 실행되게 하여 해당 페이지 (DOMAIN_USER/DOMAIN_CLIENT) 의 접근권한이(AP[2:0])이 PL0 에 해당하는 값으로 해석되도록 한다. 즉 비권한 모드로 AP 가 해석되도록 한다.

* !CONFIG_CPU_USE_DOMAINS  
유저페이지의 접근권한을 커널이 동일하게 가지게 된다.  
`t` suffix가 명령어에 확장되지 않으며 명령은 권한모드에서 실행된다.

---

### ARM Access Permissions Configuration [section-arm-access-permissions-configuration]

**2nd Level Short Translation Table Format**

	|31               12|11|10| 9|8   6|5  4| 3| 2| 1| 0|  
	| page frame number |nG|S |A2| TEX |AP10| C| B| 1|XN|

A2, A10 : AP[2:0]

**AP[2:0] access permissions control, Short-descriptor format only**

| AP[2:0] | PL1/2 | PL0  |
|---------|-------|------|
| 0 0 0   | NOAC  | NOAC |
| 0 0 1   | R/W   | NOAC |
| 0 1 0   | R/W   | RO   |
| 0 1 1   | R/W   | R/W  |
| 1 0 0   | -     | -    |
| 1 0 1   | R/O   | NOAC |
| 1 1 0   | -     | -    |
| 1 1 1   | R/O   | R/O  |

Any access not allowed at a processor mode generates Permission fault.  

NOAC : No access  

PL0 : Unprivileged access  
PL/1/2 : Privileged access  

AP[2] : Read Only  
AP[1] : User Access  
AP[0] : ??  

---

### How Linux Controls Access Permissions [section-how-linux-controls-access-permissions]
#### Setting Access Permissions Bits

PTE 를 설정하는 코드에서 AP[2:0] 이 설정되는 부분을 살펴볼 수 있다.  

	arch/arm/mm/proc-v7-2level.S:
	
		ENTRY(cpu_v7_set_pte_ext)
		...
		orr     r3, r3, #PTE_EXT_AP0 | 2

기본적으로 AP[0] = 1 을 설정하여 AP[2:1] 모델과 같은 단순한 모델을 사용하기 위함.

		...
		tst     r1, #L_PTE_RDONLY | L_PTE_DIRTY
		orrne   r3, r3, #PTE_EXT_APX

Read Only 속성이면 AP[2] = 1

		...
		tst     r1, #L_PTE_USER
		orrne   r3, r3, #PTE_EXT_AP1

User 모드이면 AP[1] = 1 하여 PL0 접근 허용

	#ifdef CONFIG_CPU_USE_DOMAINS
		@ allow kernel read/write access to read-only user pages
		tstne   r3, #PTE_EXT_APX
		bicne   r3, r3, #PTE_EXT_APX | PTE_EXT_AP0
	#endif
	...
	ENDPROC(cpu_v7_set_pte_ext)

* CPU_USE_DOMAINS == true  
DOMAIN_KERNEL에 대하여 DOMAIN_MANAGER가 사용된다. 따라서 권한 모드에서 커널페이지에 대한 접근은 permission faults를 일으키지 않는다.
PL0 이고 AP[2] == 1, 즉 Kernel/User 모두 R/O 접근권한이면 AP[2] = 0, AP[0] = 0 으로 하여 (AP[2:0] == 010) Kernel 모드(PL1/2) 에서 유저프로세스 주소 공간에 대한 접근 권한을 R/W로 설정한다.

| AP[2:0]   | PL1/2   | PL0  | L_PTE_?      |
|-----------|---------|------|--------------|
| 0 0 1     | R/W     | NOAC |              |
| 0 1 1     | R/W     | R/W  | L_PTE_USER   |
|           |         |      |              |
| 1 0 1     | R/O     | NOAC | L_PTE_RDONLY |
| **0 1 0** | **R/W** | RO   | L_PTE_USER   |

> Unsolved Question
> user R/O인 페이지에 대하여 권한모드가 왜 R/W 접근권한을 가져야 하는가?

* CPU_USE_DOMAINS != true  
DOMAIN_CLIENT 만이 사용되며 권한 모드에세서 kernel 페이지에 대한 접근도 permissions fault 를 일으킬 수 있다.  

| AP[2:0] | PL1/2 | PL0  | L_PTE_?      |
|---------|-------|------|--------------|
| 0 0 1   | R/W   | NOAC |              |
| 0 1 1   | R/W   | R/W  | L_PTE_USER   |
|         |       |      |              |
| 1 0 1   | R/O   | NOAC | L_PTE_RDONLY |
| 1 1 1   | R/O   | R/O  | L_PTE_USER   |

#### AP[2:0] v.s. AP[2:1] Model
2level tranaslation table을 이용하는 ARMv7의 시스템에서는 SCTRL.AFE 를 0 으로 설정하므로
AP[2:0]을 전부 사용하는 full AP[2:0] access permissions model 을 사용한다.
허나 `set_pte_ext()` 에서 AP0 에 대한 기본값을 1로 설정하여 두기 때문에 실제적으로는 보다 간단한 AP[2:1] 모델을 사용하게 된다. (CONFIG_CPU_USE_DOMAIN != true 인 경우에만)
