---
 layout : post
 title: init_pg_dir 도입 배경과 패치 내용 분석
 comment: true
 categories: Linux-kernel
 tags : [init_pg_dir,swapper_pg_dir]
---

## init_pg_dir, swapper_pg_dir
<br>
&nbsp; head.S의 map_memory 매크로를 하다 보면 이상한 점을 찾을 수 있다. 

	/*
	 * Map the kernel image (starting with PHYS_OFFSET).
	 */
	adrp	x0, init_pg_dir
	mov_q	x5, KIMAGE_VADDR + TEXT_OFFSET	// compile time __va(_text)
	add	x5, x5, x23			// add KASLR displacement
	mov	x4, PTRS_PER_PGD
	adrp	x6, _end			// runtime __pa(_end)
	adrp	x3, _text			// runtime __pa(_text)
	sub	x6, x6, x3			// _end - _text
	add	x6, x6, x5			// runtime __va(_end)

	map_memory x0, x1, x5, x6, x7, x3, x4, x10, x11, x12, x13, x14
	
&nbsp;  위 코드는 커널 이미지를 **init_pg_dir**에 매핑한다. 하지만 **map_memory**의 매크로는 아래의 코드와 같고, 중간에 **swapper_pg_dir**와 관련있어 보이는 전처리 구문이 있다.
		
	.macro map_memory, tbl, rtbl, vstart, vend, flags, phys, pgds, istart, iend, tmp, count, sv
		...
	#if SWAPPER_PGTABLE_LEVELS > 3
		compute_indices \vstart, \vend, #PUD_SHIFT, #PTRS_PER_PUD, \istart, \iend, \count
		populate_entries \tbl, \rtbl, \istart, \iend, #PMD_TYPE_TABLE, #PAGE_SIZE, \tmp
		mov \tbl, \sv
		mov \sv, \rtbl
	#endif
	#if SWAPPER_PGTABLE_LEVELS > 2
		compute_indices \vstart, \vend, #SWAPPER_TABLE_SHIFT, #PTRS_PER_PMD, \istart, \iend, \count
		populate_entries \tbl, \rtbl, \istart, \iend, #PMD_TYPE_TABLE, #PAGE_SIZE, \tmp
		mov \tbl, \sv
	#endif
	
&nbsp; 해당 코드를 처음 분석할때는, 저장하는 곳은 **init_pg_dir**인데 전처리 조건의 이름은 **swapper_pg**에 관련되어 매우 혼란스러웠다. 이런 불일치는 **init_pg_dir**이 도입되면서 발생한 문제이다. 해당 패치 내용을 아래에서 자세히 살펴보자.

## init_pg_dir 도입 패치 분석
<br>
&nbsp; 패치의 이름은 [Arm64/mm: Separate boot-time page tables from swapper_pg_dir](https://github.com/iamroot16/linux/commit/2b5548b68199c17c1466d5798cf2c9cd806bdaa9#diff-fa9e05a9b81e3f7eb7a1d4c18baeb77c) 이다. 먼저 아래 변경사항을 확인해 보자.

![enter image description here](https://github.com/YWHyuk/YWHyuk.github.io/blob/master/img/init_pg_dir.PNG?raw=true)

&nbsp; 위에서 봤던 코드 내용에서 변경된 내용을 확인할 수 있다. 위 패치 이전에는 swapper_pg_dir에 부트 타임 페이지 테이블을 만들었다. 이런 이유로 map_memory의 전처리 조건 이름이 swapper_pg와 관련된 것이다.

&nbsp; 그렇다면 왜 이런 패치를 수행했는지 알아보자. 앞서 말했듯이 이전 버전의 커널은 부팅 타임 페이지 테이블을 swapper_pg_dir에 생성한다. 부팅 초기 단계에는 메모리 할당자가 작동하지 않으므로, 정적으로 공간을 할당해줘야 한다.  페이징 관련 설정은 아래와 같다고 가정한다.

1. VA_BIT = 48
2. Paging = 4KB
3. Paging level = 4

&nbsp; 위와 같은 설정이라면, 총 3개의 페이지가 필요하고, 매크로에 의해 정적으로 공간이 할당된다. head.S에서 위 3개의 페이지를 사용하여 부팅 타임 페이지 테이블을 구성한다. 여러 작업이 수행된 후, paging_init 함수에서 정식으로 커널 이미지를 매핑하게 된다.  해당 과정을 한번 살펴 보자.

	phys_addr_t pgd_phys = early_pgtable_alloc();
	pgd_t *pgdp = pgd_set_fixmap(pgd_phys);

	map_kernel(pgdp);
	map_mem(pgdp);

	/*
	 * We want to reuse the original swapper_pg_dir so we don't have to
	 * communicate the new address to non-coherent secondaries in
	 * secondary_entry, and so cpu_switch_mm can generate the address with
	 * adrp+add rather than a load from some global variable.
	 *
	 * To do this we need to go via a temporary pgd.
	 */
	cpu_replace_ttbr1(__va(pgd_phys));
	memcpy(swapper_pg_dir, pgdp, PGD_SIZE);
	cpu_replace_ttbr1(lm_alias(swapper_pg_dir));
	
	pgd_clear_fixmap();
	memblock_free(pgd_phys, PAGE_SIZE);
		/*
	 * We only reuse the PGD from the swapper_pg_dir, not the pud + pmd
	 * allocated with it.
	 */
	memblock_free(__pa_symbol(swapper_pg_dir) + PAGE_SIZE,
		      __pa_symbol(swapper_pg_end) - __pa_symbol(swapper_pg_dir)
		      - PAGE_SIZE);

* line 1. 메모리 할당자를 통해 PGD 테이블을 동적할당 받는다.
* line 2. 할당 받은 물리 메모리를 fixmap 영역에 매핑시키고, 매핑된 가상 주소를 리턴받는다.
* line 4~5.  커널 이미지를 정식 매핑한다. 위 과정에서 하위 페이지 테이블이 생성된다.
* line 16,19. swapper_pg_dir의 PGD 페이지 테이블을 재사용한다. 

&nbsp; 내용을 정리하면, 정식 매핑 과정에서는 메모리 할당자를 사용할 수 있기에 페이지 테이블을 만들기 위한 공간은 동적으로 할당받는다. 페이지 테이블이 구성된 후, swapper_pg_dir의 첫 페이지만 재사용한다.  swapper_pg_dir은 rodata 섹션에 위치해있기 때문에, map_mem 함수에서 사용되지 않는 2페이지도 rodata 섹션에 묶여버리게 된다.

![enter image description here](https://github.com/YWHyuk/YWHyuk.github.io/blob/master/img/swapper_pg_dir.png?raw=true)

&nbsp; 결론은 다음과 같다. 부팅 단계에 구성한 페이지 테이블은 한 페이지를 제외하고 나머지는 부팅 이후에 사용되지 않는다.  따라서 이 사용되지 않는 영역을 init섹션으로 배치하는게 위 패치의 주요 목적이다. 변경 사항은 크게 세 가지이다.

* 부팅 타임 페이지 테이블은 init_pg_dir으로 구성한다. 부팅 초기 단계의 swapper_pg_dir이 수행한 역활과 동일하다. 단, 존재하는 섹션의 위치가 다르다.
* swapper_pg_dir은 한 페이지만 정적으로 할당 받고, paging_init()에서 복사되며, ttbr에 등록된다.
* INIT_MM_CONTEXT라는 매크로가 선언되며, 초기 메모리 디스크립터의 pgd 멤버를 init_pg_dir로 재설정한다.
