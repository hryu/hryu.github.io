---
layout: post
title: Setting Page Table Entry on ARMv7
---

## Contents

1. [Introduction](#1.-introduction)
2. [Linux Version Format](#2.-linux-version-format)
3. [ARM Version Format](#3.-arm-version-format)
4. [Setting Page Table Entry](#4-setting-page-table-entry)

---

## 1. Introduction
ARM 리눅스에서 사용하는 페이지 테이블 엔트리포맷에 대해 알아보고 이에 따라
어떻게 ARM 페이지 테이블 엔트리에 설정되게 되는지 알아본다.

> **NOTE**  
> 본 문서는 32bit ARM, 2 level, 4K, short page table descriptor 에 대한 분석
이며 리눅스에서 사용되는 설정만 설명함.  
> 보다 자세한 사항은 ARMv7-AR Reference Manual 의 B3.5.1 Short-descriptor
translation table format descriptors 을 참고.  

---

## 2. Linux Version Format

**Normal (F == 0 and P == 1)**  

    |31                 12|11|10| 9| 8| 7| 6| 5        2| 1| 0|
    | page frame number   | N| S| X| U| R| D| MT        | Y|P/V

    N : L_PTE_NONE
    S : L_PTE_SHARED
    X : L_PTE_XN
    U : L_PTE_USER
    R : L_PTE_RDONLY
    D : L_PTE_DIRTY
    MT:
     00 : L_PTE_MT_UNCACHED
     01 : L_PTE_MT_BUFFERABLE
     02 : L_PTE_MT_WRITETHROUGH
     03 : L_PTE_MT_WRITEBACK
     06 : L_PTE_MT_MINICACHE
     07 : L_PTE_MT_WRITEALLOC
     04 : L_PTE_MT_DEV_SHARED
     0C : L_PTE_MT_DEV_NONSHARED
     09 : L_PTE_MT_DEV_WC
     0B : L_PTE_MT_DEV_CACHED
    Y : L_PTE_YOUNG
    P : L_PTE_PRESENT or L_PTE_VALID

**Non-linear (F == 1)**  

    |31                                             3| 2| 1| 0|
    | file page offset                               | 1| 0| 0|

**Swap	(F == 0 and P == 0)**  

    |31                                      8| 7   3| 2| 1| 0|
    | offset                                  | type | 0| 0| 0|

---

## 3. ARM Version Format

    |31                     12|11|10| 9|8   6|5  4| 3| 2| 1| 0|
    | page frame number       |nG|S |A2| TEX |AP10| C| B| 1|XN|

nG          : not global  
S           : Shareable  
A2, A10     : access permission  
TEX[0],C,B  : memory region attributes  
XN          : execute never  

---

## 4. Setting Page Table Entry

리눅스 버전의 pte 를 저장한 후, 이를 참고하여 arm hw 버전의 pte를 만들어
설정하는 `set_pte_at()` 함수의 ARMv7 구현을 알아본다.

    arch/arm/mm/proc-v7-2level.S:

    /*
     *      cpu_v7_set_pte_ext(ptep, pte)
     *
     *      Set a level 2 translation table entry.
     *
     *      - ptep  - pointer to level 2 translation table entry
     *                    (hardware version is stored at +2048 bytes)
     *      - pte   - PTE value to store
     *      - ext   - value for extended PTE bits
     */
    ENTRY(cpu_v7_set_pte_ext)
    #ifdef CONFIG_MMU
        str     r1, [r0]                        @ linux version

리눅스 버전의 pte 값을 바로 저장  

        bic     r3, r1, #0x000003f0
        bic     r3, r3, #PTE_TYPE_MASK
        orr     r3, r3, r2

0x00003f0       : TEX[2..0], AP[1,0] clear  
PTE_TYPE_MASK   : 하위 2 비트 clear, 1x 일때 2level 4k pte를 의미  
r2              : ext 비트 set, set_pte_at()을 사용하여 진입할때 유저프로세스
                  공간의 경우 PTE_EXT_NG 를 셋함.   

        orr     r3, r3, #PTE_EXT_AP0 | 2

PTE_EXT_AP0     : AP[0]에 1을 기본 설정함.  
2               : 4k page 사용  

        tst     r1, #1 << 4
        orrne   r3, r3, #PTE_EXT_TEX(1)

1 << 4          : memory type 체크, memory type 이 2 비트부터이므로
                  MT & (1 << 2)로 해석 가능. 즉 캐시가 켜지는 설정의 경우,
                  TEX[0] 에 1을 셋, 아니면 클리어  

0 : 0000 : L_PTE_MT_UNCACHED  
1 : 0001 : L_PTE_MT_BUFFERABLE  
**2 : 0010 : L_PTE_MT_WRITETHROUGH**  
**3 : 0011 : L_PTE_MT_WRITEBACK**  
**6 : 0110 : L_PTE_MT_MINICACHE**  
**7 : 0111 : L_PTE_MT_WRITEALLOC**  
4 : 0100 : L_PTE_MT_DEV_SHARED  
C : 1100 : L_PTE_MT_DEV_NONSHARED  
9 : 1001 : L_PTE_MT_DEV_WC  
**B : 1011 : L_PTE_MT_DEV_CACHED**  

PTE_EXT_TEX(1)  : TEX[0] = 1, TEX[0],C,B 는 PRRR, NMRR 의 인덱스로 사용되는데
                  TEX[0]이 1이므로 가능한 TEX0,C,B 인덱스는 4,5,7 임
                  (6은 사용되지 않음). 또한 C,B 는 MT 의 하위 2비트가 그대로
                  사용됨. WRITETHOUGH와 MINICACHE는 v7에서 사용되지 않음.  

| L_PTE_MT_*    | TEX[0],C,B | TR       | I/OR                      | NOS(1) |
| ------------- | ---------- | -------- | ------------------------- | ------ |
| UNCACHED      | 000        | S.O.     | -                         |       1|
| BUFFERABLE    | 001        | Normal   | None-cacheable            |       1|
| WRITETHROUGH  | 110        | -        | -                         |       1|
| WRITEBACK     | 111        | Normal   | Write-Back, Write-Allocate|       1|
| MINICACHE     | 110        | -        | -                         |       1|
| WRITEALLOC    | 111        | Normal   | Write-Back, Write-Allocate|       1|
| DEV_SHARED    | 100        | Device   | -                         |       1|
| DEV_NONSHARED | 100        | Device   | -                         |       1|
| DEV_WC        | 001        | Normal   | None-cacheable            |       1|
| DEV_CACHED    | 111        | Normal   | Write-Back, Write-Allocate|       1|

S.O.    : Strongly-ordered  

        eor     r1, r1, #L_PTE_DIRTY
        tst     r1, #L_PTE_RDONLY | L_PTE_DIRTY
        orrne   r3, r3, #PTE_EXT_APX

dirty bit 에뮬레이션을 위한 것으로서 `L_PTE_DIRTY` 가 꺼져 있거나
`L_PTE_RDONLY` 가 켜져있다면 `PTE_EXT_APX` 비트를 켜서 해당 페이지를 read-only
로 만든다. 이로서 `L_PTE_DIRTY`가 꺼진 페이지에 쓰기를 시도하면 page fault가
일어나게 된다.  

        tst     r1, #L_PTE_USER
        orrne   r3, r3, #PTE_EXT_AP1
    #ifdef CONFIG_CPU_USE_DOMAINS
        @ allow kernel read/write access to read-only user pages
        tstne   r3, #PTE_EXT_APX
        bicne   r3, r3, #PTE_EXT_APX | PTE_EXT_AP0
    #endif

L_PTE_USER  : 이 비트가 세팅되면 PTE_EXT_AP1, 즉 AP[1] = 1.  
PTE_EXT_AP1 : 세팅되면 비특권 모드에서의 페이지 접근을 허락함.  

CONFIG_CPU_USE_DOMAINS  == 1 일때,  

| AP[2:0]   | PL1/2   | PL0  | L_PTE_?      |
|-----------|---------|------|--------------|
| 0 0 1     | R/W     | NOAC |              |
| 0 1 1     | R/W     | R/W  | L_PTE_USER   |
|           |         |      |              |
| 1 0 1     | R/O     | NOAC | L_PTE_RDONLY |
| **0 1 0** | **R/W** | RO   | L_PTE_USER   |

 User 모드에 사용되는 PTE이며 APX 가 셋된 경우, 즉 AP[2:0] == 111 이면
kernel/user 모두 R/O 가 되며 이경우 AP[2]와 AP[0]을 클리어하여 AP[2:0| == 010,
즉 kernel 모드(PL1/2)에서는 해당 페이지에 대한 (유저 주소공간에 속해 있다.)
접근 권한이 R/W가 되도록 한다.  
>NOTE  
유저 프로세스가 커널모드로 진입하여 유저스페이스 주소공간에 대한 접근을 시도하
는 경우를 생각해보라. 이때 프로세서는 PL1/2 모드이며 해당 페이지의 domain는
user/client로 설정하였으므로 AP 설정을 체크하게 된다.  

CONFIG_CPU_USE_DOMAINS == 0 일때,  

| AP[2:0]   | PL1/2   | PL0  | L_PTE_?      |
|-----------|---------|------|--------------|
| 0 0 1     | R/W     | NOAC |              |
| 0 1 1     | R/W     | R/W  | L_PTE_USER   |
|           |         |      |              |
| 1 0 1     | R/O     | NOAC | L_PTE_RDONLY |
| **1 1 1** | **R/O** | R/O  | L_PTE_USER   |

        tst     r1, #L_PTE_XN
        orrne   r3, r3, #PTE_EXT_XN

실행가능한 페이지인지 여부에 따라 설정한다.  

        tst     r1, #L_PTE_YOUNG
        tstne   r1, #L_PTE_VALID
    #ifndef *CONFIG_CPU_USE_DOMAINS*
        eorne   r1, r1, #L_PTE_NONE
        tstne   r1, #L_PTE_NONE
    #endif
        moveq   r3, #0

access bit 에뮬레이션을 위한 것으로서 `L_PTE_YOUNG` 비트가 꺼진 pte 설정하려
시도할때 H/W pte 의 값을 0으로 리셋하여 설정함으로서 해당 페이지에 접근하였을때
page fault가 일어나도록 만든다. 추가적으로 !CPU_USE_DOMAIN(armv7에서 일반적)
일때 L_PTE_NONE 이 켜져있다면 pte 값을 0으로 리셋한다.  

     ARM(   str     r3, [r0, #2048]! )
     THUMB( add     r0, r0, #2048 )
     THUMB( str     r3, [r0] )

ARM H/W 버전의 PTE를 저장한다.  

        mcr     p15, 0, r0, c7, c10, 1          @ flush_pte

clean L1 d-cache for cache <-> memory coherency  

    #endif
        mov     pc, lr
    ENDPROC(cpu_v7_set_pte_ext)
