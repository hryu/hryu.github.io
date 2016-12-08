---
layout: post
title: Linux Memory Types and ARM Memory Region Attributes
---

## Contents

1. [Introduction][section-introduction]
2. [Linux PTE Memory Type and TEX remap index][section-memory-type-tex-remap]
3. [TEX remap index and PRRR, NMRR][section-tex-remap-prrr-nmrr]

---

## 1. Introduction[section-introduction]
리눅스에서 사용하는 메모리 타입에 따른 ARMv7 에서의 메모리 타입이 어떻게 매핑
되는지 알아본다.

>NOTE  
>본 문서는 32bit ARM, 2 level page table, 4k page, short page table descriptor
>에 대한 분석이며 리눅스에서 사용되는 설정만 설명한다.  
>보다 자세한 사항은 ARMv7-AR Reference Manual 의  
> 1. A3.5 Memory types and attributes and the memory order model  
> 2. B3.5.1 Short-descriptor translation table format descriptors  
> 3. B3.8.3 Short-descriptor format memory region attributes, with TEX remap  
> 4. B4.1.110 NMRR, Normal Memory Remap Register, VMSA  
> 5. B4.1.127 PRRR, Primary Region Remap Register, VMSA  
>을 참고하라.  

---

## 2. Linux PTE Memory Type and TEX remap index [section-memory-type-tex-remap]

| Hex   | Binary | Memory Type                  |
| ----- | ------ | ---------------------------- |
| 0     | 0000   | L_PTE_MT_UNCACHED		    |
| 1     | 0001   | L_PTE_MT_BUFFERABLE		    |
| 4     | 0100   | L_PTE_MT_DEV_SHARED		    |
| C     | 1100   | L_PTE_MT_DEV_NONSHARED	    |
| 9     | 1001   | L_PTE_MT_DEV_WC  		    |
| **2** | **0010** | **L_PTE_MT_WRITETHROUGH**  |
| **3** | **0011** | **L_PTE_MT_WRITEBACK**     |
| **6** | **0110** | **L_PTE_MT_MINICACHE**     |
| **7** | **0111** | **L_PTE_MT_WRITEALLOC**    |
| **B** | **1011** | **L_PTE_MT_DEV_CACHED**    |

캐쉬가 켜지게 되는 설정의 경우 (if MT[1] == 1), `set_pte_ext()` 에서 TEX[0] = 1
이 할당되며 하위 2비트는 그대로 사용됨.  
즉, TEX[0],C,B == [TEX[0] | MT[1] | MT[0]]  

| L_PTE_MT_*    | TEX[0],C,B | TR       | I/OR                          |
| ------------- | ---------- | -------- | ----------------------------- |
| UNCACHED      | 000        | S.O.     | -                             |
| BUFFERABLE    | 001        | Normal   | None-cacheable                |
| WRITETHROUGH  | 110        | -        | -                             |
| WRITEBACK     | 111        | Normal   | Write-Back, Write-Allocate    |
| MINICACHE     | 110        | -        | -                             |
| WRITEALLOC    | 111        | Normal   | Write-Back, Write-Allocate    |
| DEV_SHARED    | 100        | Device   | -                             |
| DEV_NONSHARED | 100        | Device   | -                             |
| DEV_WC        | 001        | Normal   | None-cacheable                |
| DEV_CACHED    | 111        | Normal   | Write-Back, Write-Allocate    |

>NOTE  
>S.O.    : Strongly-ordered  
>NOS     : 1  

---

### 3. TEX remap index and PRRR, NMRR [section-tex-remap-prrr-nmrr]

| TEX[0],C,B    | TR            | IR (== OR)                            |
| ------------- | ------------- | ------------------------------------- |
| 000           | 00 (S.O)      | -                                     |
| 001           | 10 (Normal)   | 00 (None-cacheable)                   |
| 010           | 10 (Normal)   | 10 (Write-Through, no Write-Allocate) |
| 011           | 10 (Normal)   | 11 (Write-Back, no Write-Allocate)    |
| 100           | 01 (Device)   | -                                     |
| 101           | 00 (S.O.)     | -                                     |
|               | -             | -                                     |
| 111           | 10 (Normal)   | 01 (Write-Back, Write-Allocate)       |

>NOTE  
>S.O.    : Strongly-ordered  
>NOS     : 1  
>110 은 의미없음.  

**PRRR Format**  

    |31       24|23  20|19 18|17 16|15      0|
    | NOS7...0  | SBZP |NS10 |DS10 | TR7...0 |

PRRR은 `arch/arm/mm/proc-v7-2level.S`에서 **0xff0a81a8**로 설정됨.

 NOS[7...0] == 1  
 for Normal and Device
  0 Memory region is Outer Shareable.  
  1 Memory region is Inner Shareable.  

 NS[1...0] and DS[1...0]
 | 1| 0|
 | 1| 0|

PTE의 S 비트에 따라 1번째 혹은 0번째 비트가 사용됨.  

 NS for Normal, DS for Device  
  0 Region is not Shareable  
  1 Region is Shareable.  

 TR[7...0]  
  | 7| 6| 5| 4| 3| 2| 1| 0|  
  |10|00|00|01|10|10|10|00|  

  00 Strongly-ordered.  
  01 Device.  
  10 Normal Memory.  
  11 Reserved, effect is UNPREDICTABLE.  

**NMRR Format (Only for Normal Memory)**  

    |31       16|15        0|
    |  OR7...0  |  OR7...0  |

NMRR은 PRRR과 마찬가지로 `arch/arm/mm/proc-v7-2level.S`에서 **80x40e040e0** 로
설정됨.  

 OR[7...0] and IR[7...0]
  | 7| 6| 5| 4| 3| 2| 1| 0|  
  |01|00|00|00|11|10|00|00|  

  00 Region is Non-cacheable.  
  01 Region is Write-Back, Write-Allocate.  
  10 Region is Write-Through, no Write-Allocate.  
  11 Region is Write-Back, no Write-Allocate.  
