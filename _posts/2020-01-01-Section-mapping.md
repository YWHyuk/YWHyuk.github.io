---
layout: post
title: Section mapping
comments: true
categories: Linux-kernel
tags : [paging, arm64, 5.3.18]
---

&nbsp; 커널의 첫 부분 **head.S**를 분석하다 보면,  **Page table**을 형성하는 것을 알 수 있습니다. 
**Page table**을 생성하는 과정을 따라가 보면, 섹션 매핑을 접할 수 있습니다.  

&nbsp; 분석 당시, 설정해준 페이징 설정과는 다르게 매핑되는 과정 때문에 골치 아팠던 기억이 아련하게 떠오릅니다.

&nbsp; 커널은 config파일을 통해 다양하게 설정을 변경할 수 있는데, 그 중 위 주제와 관련된 설정은 아래와 같습니다.

	CONFIG_ARM64_4K_PAGES=y
	
위 설정의 의미는 **4K Paging**을 사용한다는 의미입니다. 


[**arch/arm64/include/asm/kernel-pgtable.h**]

	#ifdef CONFIG_ARM64_4K_PAGES
	#define ARM64_SWAPPER_USES_SECTION_MAPS 1
	#else
	#define ARM64_SWAPPER_USES_SECTION_MAPS 0
	#endif
	
&nbsp; 위의 전처리 구문을 통해 **4K Paging**이 설정되어 있다면, 섹션 매핑을 사용하게 됩니다. 

[**arch/arm64/include/asm/kernel-pgtable.h**]

	#if ARM64_SWAPPER_USES_SECTION_MAPS
	#define SWAPPER_PGTABLE_LEVELS (CONFIG_PGTABLE_LEVELS - 1)
	...
	#endif
	
&nbsp; 섹션 매핑을 사용한다면, **SWAPPER_PGTABLE_LEVELS**을 기존 설정보다 한단계 낮춥니다. 위 전처리에 따라 섹션 매핑을 한다면 **head.S/map_memory** 매크로에서 한 단계의 페이지 테이블이 생성 단계가 생략됩니다.

&nbsp; 아래 그림은 섹션 매핑이 설정되었을 때 페이지 테이블이 생성되는 과정을 나타내고 있습니다. 

![map memory_diagram](https://github.com/YWHyuk/YWHyuk.github.io/blob/master/img/map_memory.png?raw=true)

&nbsp; 그렇다면 MMU는 접근하는 엔트리, 
1. **섹션 매핑**인지,
2. **다음 단계 페이지 테이블**인지,
3. **페이지 프레임**인지

구분할 수 있어야 합니다. MMU는 페이지 테이블 엔트리의 디스크립터로 이를 식별할 수 있습니다.
## Entry Descriptor

 ---
 
 &nbsp; **ArmV8 아키텍처 매뉴얼**에 따르면, 변환 레벨에 따라 엔트리 디스크립터의 포맷의 의미가 다르다고 합니다. 엔트리 디스크립터는 엔트리의 하위 2비트가 결정하며, 대응하는 변환 레벨에 따라 의미가 다릅니다.
 ### Descriptor encodings, 0 level, 1 level, 2 level format

 - Descriptor bit[0] : 해당 엔트리가 유효한지를 나타냅니다. 유효하지 않은 엔트리를 접근하게 되면 Translation fault를 발생하게 됩니다.
 - Descriptor bit[1]: 디스크립터의 타입을 식별합니다.
	 - Descriptor bit[1] = 0 : 해당 엔트리는 블록 매핑된 메모리의 베이스를 저장하고 있음을 의미합니다.
	 - Descriptor bit[1] = 1 : 해당 엔트리는 다음 단계의 페이지 테이블 주소를 저장하고 있음을 의미합니다.

### Descriptor encodings, 3 level format

 - Descriptor bit[1:0] = 0b00 : 해당 엔트리가 유효한지를 나타냅니다. 유효하지 않은 엔트리를 접근하게 되면 Translation fault를 발생하게 됩니다.
 - Descriptor bit[1:0] = 0b01 : 0b00와 같이 취급되고, level3 테이블에서는 사용되어서는 안됩니다.
 - Descriptor bit[1:0] = 0b11 : 매핑된 물리 메모리 주소를 저장하고 있음을 의미합니다.

## 장점

---
### TLB Miss
&nbsp; 그렇다면 섹션 매핑을 할 때와 섹션 매핑을 하지 않았을 때를 비교하면서 섹션 매핑을 하면서 얻는 이득을 살펴봅시다. 

&nbsp; 비교를 위해, 간단한 벤치 마크 프로그램을 떠올려봅시다. 이 벤치 마크 프로그램은 커널 이미지를 처음부터 끝까지 순차적으로 읽는 프로그램입니다. 커널 설정은 48Bit, 4K page이고, 커널 이미지는 22MB고 2MB로 정렬되어 있다고 가정합니다. 

&nbsp; 아래 그림은 48Bit, 4K page에서 가상 주소의 변환 과정을 전부 나타낸 그림입니다. 각 Lookup 과정마다 TLB가 Miss 또는 Hit할 것 입니다.
![48_4](https://github.com/YWHyuk/YWHyuk.github.io/blob/master/img/48_4K_translation.PNG?raw=true)

&nbsp;  캐시의 스펙에 따라 정확한 값은 달라지겠지만, 최소 TLB Miss count는 아래 표와 같습니다.

|                           | PGD| PUD | PMD | PTE | Total |
|---|:---:|:---:|:---:|:---:|:---:|
|  Section Mapping TLB Miss | 1 | 1  | 1  | 0  | 3|
|   Non Section Mapping TLB Miss| 1  | 1  | 1  | 11  | 14 | 

&nbsp; 이렇게 연속된 물리 메모리를 매핑할 때에는 블록 매핑을 하는게 TLB 활용에 도움이 되는 것 같습니다. 이와 비슷한 이유로 페이지 엔트리에서 **Continuous bit** 옵션을 제공합니다.
 
### Memory Size
&nbsp; 48Bit, 4K page일때, 섹션 매핑을 할 때와 섹션 매핑을 하지 않았을 때, 변환 테이블을 구성하기 위한 페이지 수를 비교하면서 섹션 매핑을 하면서 얻는 이득을 살펴봅시다.

|   | PGD  | PUD  | PMD  | PTE  | Total |
|---|:---:|:---:|:---:|:---:|:---:|
|  Section Mapping Needed Page | 1 | 1  | 1  | 0  | 3 |
|   Non Section Mapping Needed Page| 1  | 1  | 1  | 11  | 14|

&nbsp; 다음 단계의 테이블의 전체 매핑 크기만큼 물리 메모리가 연속되어 있다면, 메모리 활용 측면에서 섹션 매핑을 하는게 훨씬 더 합리적으로 보입니다.
 
## 관련 자료 

 - **ARMv8-A_Architecture_Reference_Manual (Chapter D5.4)** 
 - **http://jake.dothome.co.kr/pt64/**
