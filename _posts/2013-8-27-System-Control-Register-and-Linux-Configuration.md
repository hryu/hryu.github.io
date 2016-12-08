---
layout: post
title: System Control Register and Linux Configuration on ARMv7 using 2 Level Page Table
---

## Contents

1. [Introduction][section-introduction]
2. [SCTRL Configuration in Linux][section-sctrl-configuration-in-Linux]

---

## Introduction [section-introduction]

2 Level Page Table을 사용하는 ARMv7 시스템에서 SCTRL 레지스터를 어떻게 설정하고 있는지 살펴보고 그 의미를 알아본다.

>TODO  
>스펙에서 찾지못한 비트들을 클리어/셋 하고 있다. 해당 비트의 의미를 찾으시오. (include/asm/cp15.h 참고)
---

## SCTRL Configuration in Linux [section-sctrl-configuration-in-Linux]

### Setting SCTRL value

	arch/arm/mm/proc-v7-2level.S:
	
		/*   AT                                                                 
		 *  TFR   EV X F   I D LR    S                                          
		 * .EEE ..EE PUI. .T.T 4RVI ZWRS BLDP WCAM                              
	 	 * rxxx rrxx xxx0 0101 xxxx xxxx x111 xxxx < forced                     
	 	 *   01    0 110       0011 1100 .111 1101 < we want                    
	 	 */
		 .align  2
		 .type   v7_crval, #object
	 v7_crval:
		 crval   clear=0x2120c302, mmuset=0x10c03c7d, ucset=0x00c01c7c

	arch/arm/mm/proc-v7.S:
	
	__v7_setup:
		...
		adr     r5, v7_crval
		ldmia   r5, {r5, r6}
		...
		mrc     p15, 0, r0, c1, c0, 0           @ read control register
		bic     r0, r0, r5                      @ clear bits them
		orr     r0, r0, r6                      @ set them
		...
	ENDPROC(__v7_setup)

SCTLR 레지스터를 읽은 후 0x2120c302 로 clear 후, 0x10c03c7d 를 그 값에 or 함.

>NOTE  
>`__v7_setup` 은 `proc_info_list.__cpu_flush` 와 연결되며 `arch/arm/kernel/head.S` 의 `ENTRY(stext)` 에서 호출됨.

### SCTRL Configuration and Linux

	|31|30|29|28|27|26|25|24|23...18|17|16|15|14|13|12|11|10|9|8|7...3| 2| 1| 0|  
	     T  A  T  N		E  V	   	  H		   R  V  I  Z  S			C  A  M
		 E  F  R  M		E  E	      A		   R		   W
			E  E  F
				  I
	
>NOTE  
>virtically written.

#### clear 0x2120c302
clear [29],[24],[15],[14],[9],[8],[1]. this means...  

**AFE, bit[29]	: Access Flags Enable**  
	0 : AP[0] is an access permissions bit. The full range of access permissions is supported.

VE, bit[24]	: Interrupt Vectors Enable  
	0 : Use the FIQ and IRQ vectors from the vector table, see the V bit entry.

IT, bit[15]	: RAZ/SBZP  
RR, bit[14]	: Round Robin cache replacement policy  
	0 : random replacement.

S, bit[9]	: RAZ/SBZP  
B, bit[8]	: RAZ/SBZP  
A, bit[1]	: Alignment check enable  
	0 : Alignment fault checking disabled.

>NOTE  
>on cortex-9,  
>VE, bit[24] read as RAZ/WI (means ON)  
>L4, bit[15] 스펙에서 찾지 못함, RAZ/SBZP.  
>S, bit[9] 스펙에서 찾지 못함, RAZ/SBZP.  
>B, bit[8] 스펙에서 찾지 못함, RAZ/SBZP.  

#### set 0x10c03c7d
set [28],[23],[22],[13],[12],[11],[10],[6],[5],[4],[3],[2],[0]. this means...  

**TRE, bit[28]	: TEX Remap**  
 1 : TEX remap enabled

XP, bit[23] 	: RAO/SBOP.  
U, bit[22]		: RAO/SBOP.  
**V, bit[13]		: base address of exception vectors**  
 1 : high vectors (0xffff0000)

I ,bit[12]		: instruction cache  
 1 : Instruction caching enabled

Z, bit[11]		: program flow prediction  
 1 : program flow prediction enabled

SW, bit[10]		: SWP/SWPB  
 1 : SWP and SWPB peform normally.

L, bit[6]		: RAO/SBOP  
P, bit[5]		: RAO/SBOP  
D, bit[4]		: RAO/SBOP  
W, bit[3]		: RAO/SBOP  
C, bit[2]		: data cache  
 1 : Data caching enabled

M, bit[0]		: MMU enable. This is a global enable bit for the PL1&0 stage 1 MMU.
 1 : PL1&0 stage 1 MMU enabled.

>NOTE  
>on cortex-9,  
>XP, bit[23]	: 스펙에서 찾지못함. RAO/SBOP.  
>U, bit[23]		: 스펙에서 찾지못함. RAO/SBOP.  
>L, bit[6]		: 스펙에서 찾지 못함, RAO/SBOP  
>P, bit[5]		: 스펙에서 찾지 못함, RAO/SBOP  
>D, bit[4]		: 스펙에서 찾지 못함, RAO/SBOP  
>W, bit[3]		: 스펙에서 찾지 못함, RAO/SBOP  
