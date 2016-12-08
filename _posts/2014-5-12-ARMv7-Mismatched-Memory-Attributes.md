---
layout: post
title: ARMv7 Mismatched Memory Attributes.md
---

## Contents

1. [Introduction][section-introduction]  
2. [Definition][section-definition]  
3. [Possibility of Mismatched Attributes]  
   [section-possibility-of-mismatched-attributes]  
4. [Types of Mismatched Attributes][section-types-of-mismatched-attributes]  
5. [DMA Mappings][section-dma-mapping]  

---

## 1. Introduction [section-introduction]
ARMv7 에서 같은 물리 주소에 대하여 복수의 가상 주소를 생성하여 접근할 때,
다른 속성으로 생성하면 문제가 될 수 있다.

>ARMv7-AR Reference Manual A3.5.7 Memory access restrictions 참고

---

## 2. Definition [section-definition]
같은 물리 주소에 대하여 복수의 가상 주소 매핑이 동일하지 않은  
1. memory type,  
2. shareability,  
3. cacheability (cache allocation hints 제외)  
를 가지고 이루어질 때 이를 **mismatched attributes** 라 부른다.

---

## 3. Possibility of Mismatched Attributes [section-possibility-of-mismatched-attributes]

커널은 동일한 물리주소에 대하여 복수의 가상 주소 매핑을 생성하는 경우가 많은데
간단한 예로 커널에 이미 매핑된 페이지를 프로세스 주소영역에 재맵핑하거나
(그 반대거나) lowmem에 해당하는 페이지를 VMALLOC 영역에 재매핑하거나 이미 맵핑
된 페이지를 dma 오퍼레이션을 위해 재매핑하는 등의 여러 경우가 있다. 대부분의
경우 동일한 attribute 로 매핑을 생성하므로 문제가 되지는 않지만 특히 DMA를 위해
재매핑하는 경우 문제가 될 수 있다.

---

## 4. Types of Mismatched Attributes [section-types-of-mismatched-attributes]

### Mismatched Memory Type  

Strongly-ordered or Device v.s. Normal  
memory type에 의해 보장되는 속성들을 잃게 되며 그 특성들은 Normal 타입에
부가되는 특성을 갖게된다.

### Mismatched Cacheability

적절한 캐쉬 관리 정책 (clean/invalidate/flush) 으로 coherency 를 보장할 수
있다.

---

## 5. DMA Mappings [section-dma-mapping]
커널에서는 dma mapping을 위한 매핑생성시 mismatched attributes에 따른 위험을
제거하기 위하여 dma 오퍼레이션을 위한 메모리 타입으로 mismatched memory type을
유발하는 Stronly-ordered 나 Device 대신에 cacheable만 꺼진 Normal memory 타입을
사용하며 해당 페이지의 접근시 coherency를 보장하기 위한 cache management
오퍼레이션을 추가로 수행함.

    arch/arm/include/asm/pgtable.h:

    #ifdef CONFIG_ARM_DMA_MEM_BUFFERABLE
    #define pgprot_dmacoherent(prot) \
        __pgprot_modify(prot, L_PTE_MT_MASK, L_PTE_MT_BUFFERABLE | L_PTE_XN)
    #define __HAVE_PHYS_MEM_ACCESS_PROT
    struct file;
    extern pgprot_t phys_mem_access_prot(struct file *file, unsigned long pfn,
                                        unsigned long size, pgprot_t vma_prot);
    #else
    #define pgprot_dmacoherent(prot) \
        __pgprot_modify(prot, L_PTE_MT_MASK, L_PTE_MT_UNCACHED | L_PTE_XN)
    #endif

`CONFIG_ARM_DMA_MEM_BUFFERABLE`은 v7+ 에서 강제로 활성화되며 Kconfig 에서
다음과 같이 언급되고 있다.

    arch/arm/mm/Kconfig

    config ARM_DMA_MEM_BUFFERABLE
        ...
        Historically, the kernel has used strongly ordered mappings to
        provide DMA coherent memory.  With the advent of ARMv7, mapping
        memory with differing types results in unpredictable behaviour,
        so on these CPUs, this option is forced on.

        Multiple mappings with differing attributes is also unpredictable
        on ARMv6 CPUs, but since they do not have aggressive speculative
        prefetch, no harm appears to occur.

        However, drivers may be missing the necessary barriers for ARMv6,
        and therefore turning this on may result in unpredictable driver
        behaviour.  Therefore, we offer this as an option.

### `L_PTE_MT_BUFFERABLE` v.s. `L_PTE_MT_UNCACHED`

리눅스에서 사용되는 memory type과 TEX remap 에 따라  
UNCACHED 는 strongly-ordered 로  
BUFFERABLE 는 Normal memory, non-cacheable 로 매핑된다.
