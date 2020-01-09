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
	
&nbsp; 위 코드는 커널 이미지를 **init_pg_dir**에 매핑한다. 하지만 map_memory의 매크로는 아래의 코드와 같고, 중간에 **swapper_pg**의 이름을 같는 전처리 구문이 있다. 
		
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
	
&nbsp; 해당 코드를 처음 분석할때는, 저장하는 곳은 **init_pg_dir**인데 전처리 조건의 이름은  **swapper_pg**에 관련되어 매우 혼란스러웠다. 이런 불일치는 **init_pg_dir**이 도입되면서 발생한 문제이다. 해당 패치 내용을 아래에서 자세히 살펴보자.

## init_pg_dir 도입 패치 분석
<br>
&nbsp; 패치의 이름은 **[Arm64/mm: Separate boot-time page tables from swapper_pg_dir](https://github.com/iamroot16/linux/commit/2b5548b68199c17c1466d5798cf2c9cd806bdaa9#diff-fa9e05a9b81e3f7eb7a1d4c18baeb77c)**이다. 먼저 아래 변경사항을 확인해 보자.
![enter image description here](https://github.com/YWHyuk/YWHyuk.github.io/blob/master/img/init_pg_dir.PNG?raw=true)
&nbsp; 위 패치 이전에는 swapper_pg_dir에 부트 타임 페이지 테이블을 만들었다. 이런 이유로 map_memory의 전처리 조건 이름이 swapper_pg로 시작했다.

&nbsp; 그럼 본격적으로 패치 내용을 이해해 보자. 이전의 커널은 swapper_pg_dir에 커널 이미지를 매핑했다. va_bit가 48비트이고, 4K page라면 부트 과정 동안 페이지 테이블을 위해 총 3개의 페이지가 사용된다. paging_init()에서 커널 이미지이 정식으로 매핑된다. 정식 매핑 과정에서 swapper_pg_dir의 첫 페이지만 재사용되고, 나머지 두 페이지는 사용되지 않는다. 해당 내용은 아래 그림과 같다.
![enter image description here](https://github.com/YWHyuk/YWHyuk.github.io/blob/master/img/swapper_pg_dir.png?raw=true)

&nbsp; 사용되지 않는 2페이지도 rodata 영역에 묶여 있으므로, 해당 페이지는 사용할 수 없게된다. 따라서 rodata 영역에 속하지 않고, 부팅 과정에서만 사용할 수 있는 공간이 필요하게 되었고 이것이 바로 init_pg_dir이다. 

&nbsp;  init_pg_dir은 패치 이전 swapper_pg_dir의 크기만큼 정적으로 할당되고, swapper_pg_dir은 한 페이지만 정적으로 할당받게 된다. 부팅 타임에는 init_pg_dir에 페이지 테이블을 구성하고, paging_init() 과정에서 swapper_pg_dir로 페이지 테이블으로 교체되며, init_pg_dir의 공간은 반환된다.
 
 &nbsp; 부팅 과정에만 요구되었던 메모리를 분리하여 메모리를 조금 더 효율적으로 쓰게한 패치라고 생각할 수 있겠다.
