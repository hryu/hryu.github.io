---
layout: post
title: How Dirty and Accessed bits are Emulated on ARM Linux
---

## Contents

1. [Introduction][section-introduction]
2. [Meanings][section-meanings]
3. [Manipulation APIs][section-manipulation-apis]
4. [Fault Preparation][section-fault-preparation]  
4.1 [Preparation for Dirty Bit][section-prepare-dirty-bit]  
4.2 [Preparation for Young Bit][section-prepare-young-bit]  
5. [Fault Handling][section-fault-handling]  
5.1 [Handling Dirty Bit][section-handle-dirty-bit]  
5.2 [Handling Young Bit][section-handle-young-bit]  
5.3 [Setting H/W Version PTE][section-set-hw-pte]  
6. [Clearing Bits][section-clearing-bits]  
6.1 [Clearing Dirty Bit][section-clear-dirty-bit]  
6.2 [Clearing Young Bit][section-clear-young-bit]  

---

## 1. Introduction [section-introduction]

Linux Kernel MM 에서 사용되는 page table entry 관련 bit 인 dirty, young
(or accessed) 는 ARM H/W MMU 에 그 속성이 존재하지 않으며 ARM Linux 에서는
이러한 비트들을 지원하기 위해서 fault 에 의한 emulation 방식으로 해당 bit 들을
s/w version pte 상에서 관리하게 된다.

>NOTE  
v7-a, 32bit, 2level page table 에 대해서만 다룬다.  

---

## 2. Meanings [section-meanings]

* dirty, 페이지에 데이터가 쓰여졌으며 아직 디스크에 쓰여지지 않았다.  
* young, 페이지가 접근되었다.

---

## 3. Manipulation APIs [section-manipulation-apis]

    arch/arm/include/asm/pgtable-2level.h:

    #define L_PTE_YOUNG             (_AT(pteval_t, 1) << 1)
    #define L_PTE_DIRTY             (_AT(pteval_t, 1) << 6)


    arch/arm/include/asm/pgtable.h:

    #define pte_dirty(pte)      (pte_val(pte) & L_PTE_DIRTY)
    #define pte_young(pte)      (pte_val(pte) & L_PTE_YOUNG)

    ....

    #define PTE_BIT_FUNC(fn,op) \
    static inline pte_t pte_##fn(pte_t pte) { pte_val(pte) op; return pte; }

    PTE_BIT_FUNC(mkclean,   &= ~L_PTE_DIRTY);
    PTE_BIT_FUNC(mkdirty,   |= L_PTE_DIRTY);
    PTE_BIT_FUNC(mkold,     &= ~L_PTE_YOUNG);
    PTE_BIT_FUNC(mkyoung,   |= L_PTE_YOUNG);

---

## 4. Fault Preparation [section-fault-preparation]

리눅스에서는 커널내부용 PTE (s/w version) 와 H/W 설정용 PTE (h/w version)가
동일한 페이지매핑에 대하여 복수 존재하며 s/w version pte를 참고하여 h/w version
의 pte를 설정한다.  
소개에서와 같이 ARM MMU 에는 dirty, accessed bit 이 존재하지 않기 때문에
s/w version pte에 관련 비트들을 선언하고 셋/클리어/테스트 (3. 절 참고) 함으로써
h/w version pte를 설정할 때, fault를 처리할 때, 등등에 사용하게 된다.  

4.1 과 4.2 에서 s/w version pte를 참고하여 h/w version의 pte를 설정하는 함수
`cpu_v7_set_pte_ext()` 를 보면서 어떻게 dirty, accessed 비트들을 설정하기
위해 fault 를 일으킬 준비를 하는지 살펴본다.

---

### 4.1 Preparation for Dirty Bit [section-prepare-dirty-bit]

    arch/arm/mm/proc-v7-2level.S:

    ENTRY(cpu_v7_set_pte_ext)
        ...
        eor     r1, r1, #L_PTE_DIRTY
        tst     r1, #L_PTE_RDONLY | L_PTE_DIRTY
        orrne   r3, r3, #PTE_EXT_APX
        ....

Translated in C

    r1 ^= L_PTE_DIRTY
    if (r1 & (L_PTE_RDONLY | L_PTE_DIRTY)) /* with L_PTE_DIRTY off */
        r3 |= PTE_EXT_APX

a. `L_PTE_DIRTY` 가 꺼져 있거나 (dirty bit emulation)  
b. `L_PTE_RDONLY` 가 켜져있다면 (원래 read-only 페이지이면)
`PTE_EXT_APX` 비트를 켜서 해당 페이지를 read-only 로 만든다.  
이로써 해당 페이지에 쓰기를 시도하면 page fault가 일어나게 된다.

---

### 4.2 Preparation for Young Bit [section-prepare-young-bit]

    arch/arm/mm/proc-v7-2level.S:

    ENTRY(cpu_v7_set_pte_ext)
            ...
            tst     r1, #L_PTE_YOUNG
            tstne   r1, #L_PTE_VALID
    #ifndef CONFIG_CPU_USE_DOMAINS
            eorne   r1, r1, #L_PTE_NONE
            tstne   r1, #L_PTE_NONE
    #endif
            moveq   r3, #0

     ARM(   str     r3, [r0, #2048]! )
            ...

Translated in C

        if (!(r1 & L_PTE_YOUNG))    /* L_PTE_YOUNG 이 꺼져있으면 */
            r3 = 0;
        else {
    #ifndef CONFIG_CPU_USE_DOMAINS
            r1 ^= L_PTE_NONE;
            if (!(r1 & L_PTE_NONE))	/* L_PTE_NONE 이 켜져있으면 */
                r3 = 0;
    #endif
        }

`L_PTE_YOUNG` 비트가 꺼진 pte를 설정하려 시도할때 H/W pte 의 값을 0으로
설정함으로서 해당 페이지에 접근하였을때 page fault가 일어나도록 만든다.
(young bit emulation)  
추가적으로 `CPU_USE_DOMAINS` 가 사용되지 않으면 (ARMv7) L_PTE_NONE 이 설정된
페이지인지 살펴보고 켜져있다면 pte 값을 0으로 설정한다.

---

## 5. Fault Handling [section-fault-handling]
### 5.1 Handling Dirty Bit [section-handle-dirty-bit]

    mm/memory.c:

    int handle_pte_fault(...., pte_t *pte, pmd_t *pmd, unsigned int flags)
    {
        pte_t entry;

        /* handling normal cases */
        entry = *pte;
        if (!pte_present(entry) {
            ...
        }
        ...

dirty and/or young bit emulation 의 경우 `L_PTE_PRESENT` 비트가 셋된 유효한
엔트리이기 때문에 해당 블럭을 통과하게 된다.

        /* handling dirty bit */
        if (flags & FAULT_FLAG_WRITE) {
            if (!pte_write(entry))
                return do_wp_page(mm, vma, address, pte, pmd, ptl, entry);
            entry = pte_mkdirty(entry);
        }
        ...

1. COW 페이지의 경우 `do_wp_page()`내에서 `L_PTE_DIRTY`가 자연스럽게 세팅됨.
2. 이미 쓰기 가능한 페이지의 경우 `L_PTE_DIRTY`만 설정됨. (dirty bit emulation)

---

### 5.2 Handling Young Bit [section-handle-young-bit]  

    mm/memory.c:

        ...
        /* handling young bit */
        entry = pte_mkyoung(entry);
        ...

`L_PTE_YOUNG`이 설정됨. (young bit emulation)

---

### 5.3 Setting H/W Version PTE [section-set-hw-pte]

    mm/memory.c:

    ...
        if (ptep_set_access_flags(vma, address, pte, entry,
                                  flags & FAULT_FLAG_WRITE)) {
            update_mmu_cache(vma, address, pte);
        } else {
            if (flags & FAULT_FLAG_WRITE)
                flush_tlb_fix_spurious_fault(vma, address);
        }
    ...

`ptep_set_access_flags()`를 통하여 수정된 s/w version pte를 반영하여 h/w
version pte 를 원래 설정대로 저장함으로써 더 이상 dirty, accessed bit 를 위한
fault가 일어나지 않도록 함.

---

## 6. Clearing Bits [section-clearing-bits]
### 6.1 Clearing Dirty Bit [section-clear-dirty-bit]

    page_mkclean()
        page_mkclean_file()
            page_mkclean_one()

`page_mkclean_one()` 내에서 `pte_mkclean()`이 호출됨.

    mm/rmap.c:

    static int page_mkclean_one(struct page *page, struct vm_area_struct *vma,
                                unsigned long address)
    {
        ...
        if (pte_dirty(*pte) || pte_write(*pte)) {
            pte_t entry;

            flush_cache_page(vma, address, pte_pfn(*pte));
            entry = ptep_clear_flush(vma, address, pte);
            entry = pte_wrprotect(entry);
            entry = pte_mkclean(entry);
            set_pte_at(mm, address, pte, entry);
            ret = 1;
        }
        ...
    }

페이지에 쓰기 오퍼레이션을 수행했으므로 캐쉬의 변경된 내용이 메모리에 반영되지
않았을 수 있기 때문에 먼저 `flush_cache_page()` 로 먼저 캐쉬 클린 작업을 수행하
여 일관성을 보장하도록한다.  
`ptep_clear_flush()` 에서는 먼저 pte 값을 0 초기화 후 tlb 를 플러쉬한다.  
`pte_wrprotect()`로 `L_PTE_RDONLY` 비트를 설정,  
`pte_mkclean()` 로 `L_PTE_DIRTY` 비트를 해제한다.  
`set_pte_at()` 하여 h/w pte 반영을 시도한다.  
`set_pte_at()` 이후에 tlb 를 플러쉬하지 않아도 되는 이유는 tlb 엔트리가
이미 플러쉬되었고 `ptep_clear_flush()` 이후로 해당 주소 접근 이력이 없으므로
tlb가 메모리에서 새 pte 값을 읽어들일 준비가 되었기 때문이다.

`L_PTE_DIRTY` 비트가 꺼져있는 상태로 h/w pte 설정을 시도하고 있기 때문에 4.1절
에서 살펴본바와 같이 실제 h/w pte는 read-only 로 설정되어 쓰기 접근시 dirty bit
emulation을 위한 fault trap이 설정된다.

---

### 6.2 Clearing Young Bit [section-clear-young-bit]

    reclaim_clean_pages_from_list()
        or
    shrink_inactive_list()
        shrink_page_list()
            page_check_references()
                page_referenced()

        or

    shrink_active_list()
        page_referenced()

young 비트는 `page_referenced()` 함수 내에서 클리어되며 해당 함수는 페이지 회수
시 lru 리스트가 스캔될 때 호출된다.

    page_referenced()
        page_referenced_one()
            ptep_clear_flush_young_notify()
                ptep_clear_flush_young()

마지막 함수를 자세히 보자.

	mm/pgtable-generic.c:

    int ptep_clear_flush_young(struct vm_area_struct *vma,
                               unsigned long address, pte_t *ptep)
    {
        int young;
        young = ptep_test_and_clear_young(vma, address, ptep);
        if (young)
            flush_tlb_page(vma, address);
        return young;
    }

`ptep_test_and_clear_young()` 함수가 h/w pte 값을 변경하기 때문에 tlb가 해당
엔트리를 메모리로부터 다시 읽어들이도록 하기 위해 `flush_tlb_page()` 이 필요하
다. 그렇지 않으면 tlb는 stale 주소변환을 가지고 동작하게 되어 young bit
emulation을 위한 fault가 일어나지 못 한다.  
pte 재설정전에 cache 를 플러쉬할 필요가 없는 이유는 virt->phy 변환값은 유지되고
young 비트만 클리어되는 것으로 주소매핑의 변화에 따른 cache 일관성문제가 없기
때문이다. ARM 에서는 실제적으로 pte값이 결국 0으로 초기화된 후 tlb가 플러쉬된
상황 (유효한주소매핑 -> 폴트매핑) 이지만 해당 가상주소가 다른 물리페이지로 매핑
되는 것이 아니라 해당 주소 접근시 폴트핸들러에서 결국 다시 동일한 주소변환으로
pte가 설정되기 때문이다.

    include/asm-generic/pgtable.h:

    static inline int ptep_test_and_clear_young(struct vm_area_struct *vma,
                                                unsigned long address,
                                                pte_t *ptep)
    {
        pte_t pte = *ptep;
        int r = 1;
        if (!pte_young(pte))
            r = 0;
        else
            set_pte_at(vma->vm_mm, address, ptep, pte_mkold(pte));
        return r;
    }

`pte_mkold()` 가 `L_PTE_YOUNG` 를 클리어하며 4.2 절에서 살펴본바와 같이
`L_PTE_YOUNG` 설정되지 않은 s/w version pte 를 h/w pte에 설정하려고 시도할때
해당 pte는 0으로 설정된다. 즉 해당 페이지 접근시 young bit emulat ion을 위한
fault trap이 설정된다.
