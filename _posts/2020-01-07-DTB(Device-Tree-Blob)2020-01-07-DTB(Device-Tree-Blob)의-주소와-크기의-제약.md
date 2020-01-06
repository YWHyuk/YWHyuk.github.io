---
layout: post
title: DTB(Device Tree Blob) 주소와 크기에 대한 제약
comments: true
categories: Linux-kernel
tags : [DTB, FDT, Fixmap, Bootloader]
---
# DTB(Device Tree Blob) 주소와 크기에 대한 제약

오늘은 부트로더가 DTB를 물리 메모리에 올릴때 지켜야할 몇 가지 제약들과, 그 이유에 대해서 살펴볼 예정이다. 

## 디바이스 트리란
&nbsp; 디바이스 트리란, 특정 아키텍처에 대한 하드웨어 요소를 기술한 자료 구조이다. 이름과 같이 트리 형태의 형태를 가지고 있다.
 예제 소스
  flattened와 unflattended
  
##  Booting.txt
kernel documentation에서 찾을 수 있는 Booting.txt는 부팅 과정 이전에 부트로더가 수행하는 일들을 쉽게 설명한 문서이다. 이 문서에 따르면 디바이스 트리와 관련된 제약은 크게 다음 두 가지이다.
1. 2MB보다 작아야한다.
2. 올라갈 주소는 2MB로 Align되어 있어야 한다.

&nbsp; 먼저, 첫 번째 이유는 **Fixmap**과 관련이 있다. 더 자세한 내용은 다음 챕터에서 알아보자. 두 번째 이유는 [**섹션 매핑**](https://ywhyuk.github.io/linux-kernel/2020/01/01/Section-mapping.html)을 하기 위함이다. 

이전에는 2MB Align이였다면, 지금은 8Byte Align이면 된다.
## Fixmap 영역에서의 FDT

## fixmap_remap_fdt 분석

작성 중...
