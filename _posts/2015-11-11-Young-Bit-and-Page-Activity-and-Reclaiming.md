---
layout: post
title: Young Bit and Page Activity and Reclaiming
---

## Contents

1. [Introduction][section-introduction]  
2. [Accessed Bit on PTE][section-accessed-bit]  
2.1 [Setting/Clearig Accessed bit][section-set-clear-bit]  
3. [Page Activity][section-page-activity]  
3.1. [Referenced and Active Flags][section-page-flags]  
3.2. [Setting/Clearing Referenced/Active Flag]
    [section-setting-clearing-referenced-active]  
3.3. [LRU List][section-lru-list]  

---

## 1. Introduction [section-introduction]  

`pte_mkyoung()` 으로 pte 에 accessed bit가 설정되는 것은 프로세스 주소공간 접근
에 대해서 동작하며 (mapped pages) `mark_page_accessed()` 로 페이지 구조체의
플래그에 `PG_referenced` 가 설정되는 것은 I/O 읽기/쓰기 흐름에 대하여 동작한다
(unmapped pages). 이 플래그들은 별개로 설정되지만 페이지 회수시에 적절히 참조된
다.

>**TODO**  
>PG_active 를 포함하여 본문 작성 ?

---

## 2. Accessed Bit on PTE [section-accessed-bit]  
### 2.1 Setting/Clearig Accessed bit [section-set-clear-bit]  
#### Setting Accessed bit
아키텍쳐에서 지원한다면 해당 매핑을 접근할때 MMU 에서 관련 비트를 설정하준다. 지원하지 않는 아키텍쳐라면 에뮬레이션 방식을 통해 accessed bit 를 흉내낼 수 있다.  
ARM Linux 에서는 새 매핑을 설정할때 에뮬레이션방식을 통해 linux version pte 에 `L_PTE_YOUNG` 플래그를 설정한다. `L_PTE_YOUNG` 플래그가 꺼진 값을 pte 에 설정하려고 하는 경우 hw version pte 의 값을 실제로는 0으로 설정하여 폴트를 일으키게 준비해두며 이후 연속된 접근에서 폴트가 일어나면 폴트 핸들러에서 linux version pte 에 `L_PTE_YOUNG`을 설정한 후 hardware version의 pte를 본래대로 설정해두는 방식으로 accessed bit 를 에뮬레이션한다.

#### Clearing Accessed bit

accessed bit 는 페이지 회수 흐름에서 `page_referenced()` 함수에 의해 해제된다.

	shrink_active_list()
		page_referenced()

	or

	shrink_inactive_list()
		shrink_page_list()
			page_check_references()
				page_referenced()

`page_referenced()` 는 해당 페이지가 프로세스 주소공간에 매핑되어 있을 경우 리버스 매핑을 워킹하면서 페이지테이블엔트리에 accessed bit 가 설정된 횟수를 세고 이를 리턴한다.

	page_referenced()
		page_referenced_one()
			ptep_clear_flush_young_notify()
				ptep_clear_flush_young()
					ptep_test_and_clear_young();

accessed bit 의 설정 유무는 `ptep_test_and_clear_young()` 에서 확인하게 되며 accessed bit 가 켜진 경우 해제도 함께 한다. 이로 인해 이후에 해당 페이지 접근시 다시 accessed bit가 켜지도록 한다.

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

---

## 3. Page Activity [section-page-activity]  

`mark_page_accessed()` 및 pte 매핑을 통한 페이지 접근과 (<==>) `page_check_refereces()` 및 `page_referenced()` 의 호출 빈도에 따라 페이지의 상태가 결정된다.

* referenced : pte 를 통한 접근
* accessed : 일반 r/w 를 통한 접근

### mark_page_accessed() state transision

| accessed   | active   | -> | accessed     | active   |
|------------|----------|----|--------------|----------|
| unaccessed | inactive |    | *accessed*   | inactive |
| accessed   | inactive |    | *unaccessed* | *active* |
| unaccessed | active   |    | *accessed*   | active   |

### page_check_references() state transition

| referenced      | accessed   | -> | referenced   | accessed   | page_references |
|-----------------|------------|----|--------------|------------|-----------------|
| unreferenced    | unaccessed |    | unreferenced | unaccessed | *reclaim*       |
| unreferenced    | accessed   |    |              | unaccessed | *clean reclaim* |
| unreferenced^1^ | accessed   |    |              |            | *reclaim*       |
| referenced^1^   | don't care |    |              | unaccessed | active          |
| referenced = 1  | unaccessed |    |              | accessed   | keep            |
| referenced = 1  | accessed   |    |              | accessed   | active          |
| referenced > 1  | don't care |    |              | accessed   | active          |

 각주^1^ : swapbacked

>NOTE  
>starting out *inactive*  
>clean reclaim -> reclaim the page only when it's clean and not swap backed (dirty and mapped page can not be a clean reclaim candidate).  
>referenced > 1 pages are promoted to active state  

### 3.1. Referenced and Active Flags [section-page-flags]  

	include/linux.h:
	
	enum pageflags {
		...
		PG_reference
		...
		PG_active
		...
	}
	...
	PAGEFLAG(Referenced, referenced) TESTCLEARFLAG(Referenced, referenced)
	PAGEFLAG(Active, active) __CLEARPAGEFLAG(Active, active) TESTCLEARFLAG(Active, active)
	...
 
매크로 확장에 의해 다음의 매크로들이 선언됨.

	[Test/Set/Clear/TestClear]PageReferenced()
	[Test/Set/Clear/__Clear/TestClear]PageActive()

---

### 3.2. Setting/Clearing Referenced/Active Flag [section-setting-clearing-referenced-active]  
#### accessed bit on PTE for mapped pages
HW 가 혹은 에뮬레이션 방식으로 어떤 매핑에 대한 접근이 있을 때 accessed bit 를 설정해준다.

#### mark_page_accessed()

	do_generic_file_read()
	or
	generic_perform_write()
		mark_page_accessed()

`mark_page_accessed()` 는 일반적인 R/W 제어흐름에서 호출되며 `PG_referenced`와 `PG_active`를 조작한다.  
상태전이에 대한 내용은 위의 3.Page Activity 참고.

>NOTE
>이외에도 filesystem 구현 내부 및 다른 곳에서 사용되지만 이는 논외.

	mm/swap.c:
	
	void mark_page_accessed(struct page *page)
	{
			if (!PageActive(page) && !PageUnevictable(page) &&
							PageReferenced(page) && PageLRU(page)) {
					activate_page(page);
					ClearPageReferenced(page);
			} else if (!PageReferenced(page)) {
					SetPageReferenced(page);
			}
	}

#### page_check_references()

	shrink_inactive_list()
		shrink_page_list()
			page_check_references()
				page_referenced()

`page_check_references()` 는 회수 흐름에서 호출되며 accessed bit 와 `PG_referenced` 플래그를 조작한다. 함수 내에서는 아니지만 결과적으로 `PG_active` 플래그 조작에도 영항을 미친다.  
상태전이에 대한 내용은 위의 3.Page Activity 참고.

	mm/vmscan.c:
	
	static enum page_references page_check_references(struct page *page,
													  struct scan_control *sc)
	{
		int referenced_ptes, referenced_page;
		unsigned long vm_flags;

		referenced_ptes = page_referenced(page, 1, sc->target_mem_cgroup,
										  &vm_flags);
		referenced_page = TestClearPageReferenced(page);
		
		if (vm_flags & VM_LOCKED)
			return PAGEREF_RECLAIM;
		
		if (referenced_ptes) {
			if (PageSwapBacked(page))
				return PAGEREF_ACTIVATE;
			
			SetPageReferenced(page);
			
			if (referenced_page || referenced_ptes > 1)
				return PAGEREF_ACTIVATE;
			
			if (vm_flags & VM_EXEC)
				return PAGEREF_ACTIVATE;

			return PAGEREF_KEEP;
		}

		if (referenced_page && !PageSwapBacked(page))
			return PAGEREF_RECLAIM_CLEAN;
			
		return PAGEREF_RECLAIM;
	}

`page_check_references()`는 내부적으로 `page_references()`를 호출하여 pte에 설정된 accessed bit 를 해제하고 `page_referenced()`를 통한 accessed bit 참조횟수가 0 이 아닌 경우에만 `SetPageReferenced()` 로 
다시 `PG_referenced` 플래그를 설정한다. 만일 accessed bit 참조횟수가 0인 경우에는 `PG_referenced` 플래그도 해제된다.

---

### 3.3. LRU List [section-lru-list]  

페이지들은 anonoymous/file/inactive/active/unevictable 여부에 따라 각기 다른 리스트상에 존재하게 되며 lru 리스트들은 memcg 가 사용되지 않을때는 zone 당 하나씩 혹은 memcg 가 사용될 때는 memcg zone 당 하나씩 존재함. memcg 가 사용될 때 회수는 각 memcg zone 당 lru가 서로 분리 되어 있으므로 각 memcg 에 대하여 같은 비율의 페이지 회수가 들어가게 된다.
