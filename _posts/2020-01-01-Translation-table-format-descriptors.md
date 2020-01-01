---
layout: post
title: Translation table format descriptors
comments: true
categories: Demo
---

[linux.5.3.18 based on Arm64] 
커널의 첫 부분 head.S를 분석하다 보면,  Page table을 형성하는 것을 알 수 있습니다. 
Page table을 생성하는 과정을 보다보면, 섹션 매핑(블록 매핑)을 접할 수 있습니다. 

	CONFIG_ARM64_4K_PAGES=y
만약 4K Paging이 설정되어 있다면, 섹션 매핑을 사용하게 됩니다.

arch/arm64/include/asm/kernel-pgtable.h
	
	#ifdef CONFIG_ARM64_4K_PAGES
	#define ARM64_SWAPPER_USES_SECTION_MAPS 1
	#else
	#define ARM64_SWAPPER_USES_SECTION_MAPS 0
	#endif
	
섹션 매핑을 사용한다면, SWAPPER_PGTABLE_LEVELS을 기존 설정보다 한단계 낮춥니다.

	#if ARM64_SWAPPER_USES_SECTION_MAPS
	#define SWAPPER_PGTABLE_LEVELS (CONFIG_PGTABLE_LEVELS - 1)
	...
	#endif
	
따라서 섹션 매핑을 한다면 map_memory 매크로에서 한단계의 페이지 테이블이 생략됩니다. 

![map memory_diagram](https://github.com/YWHyuk/YWHyuk.github.io/blob/master/img/map_memory.png?raw=true)

블록 매핑을 통해 PMD의 한 엔트리는 2MB만큼 매핑하게 됩니다.
