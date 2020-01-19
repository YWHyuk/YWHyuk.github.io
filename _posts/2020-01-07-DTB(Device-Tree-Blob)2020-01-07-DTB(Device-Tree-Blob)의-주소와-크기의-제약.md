---
layout: post
title: DTB(Device Tree Blob) 주소와 크기에 대한 제약
comments: true
categories: Linux-kernel
tags : [DTB, FDT, Fixmap, Bootloader]
---
오늘은 부트로더가 DTB를 물리 메모리에 올릴때 지켜야할 몇 가지 제약들과, 그 이유에 대해서 살펴볼 예정이다. 
  
##  Booting.txt
<br>
&nbsp; **Booting.txt**는 부팅 과정 이전에 부트로더가 수행하는 일들을 쉽게 설명한 문서이다. 이 문서는 kernel documentation에서 찾을 수 있다. 

&nbsp; 이 문서에는 디바이스 트리 관련하여 부트로더가 지켜야할 몇가지 규칙이 있다. 이 규칙은 [**arm64: use fixmap region for permanent FDT**]([https://github.com/iamroot16/linux/commit/61bd93ce801bb6df36eda257a9d2d16c02863cdd#diff-287d2c62df40ba9d834f5994a084d060R377](https://github.com/iamroot16/linux/commit/61bd93ce801bb6df36eda257a9d2d16c02863cdd#diff-287d2c62df40ba9d834f5994a084d060R377)) 커밋을 기준으로 크게 변경되었다. 변경된 항목들은 다음과 같다.
#### 커밋 이전

 1. 8Byte 단위로 정렬되어 있어야 한다.
 2. 커널 이미지 시작으로부터 512MB 이내에 있어야 한다.
 3. 2MB 경계선(Boundary)를 넘어서는 안된다.
 4. 2MB보다 작아야 한다.

<br>
#### 커밋 이후
1. 8Byte 단위로 정렬되어 있어야 한다.
2. 2MB보다 작아야 한다.

&nbsp; 커밋 이후로, 2번과 3번 항목이 사라졌다. 먼저 이 커밋 내용을 설명하면서 2번 항목이 사라진 이유에 대하 먼저 알아보자. 

## arm64: use fixmap region for permanent FDT 
<br>
&nbsp; 커밋 이름만 봐도 유추할 수 있듯이, 해당 커밋은 **Fixmap**에 **FDT 영역**을 추가하고 FDT를 **swapper_pg_dir**(커널의 이미지를 매핑한 페이지 테이블)에 매핑하는 대신에 ,활성화된 **Fixmap**영역에 매핑시키도록 하는 내용이다. 
### 1. Fist 512MB region rule
&nbsp; 기존의 방식은 이미 만들어진 페이지 테이블을 이용하는 방식이기에, 새로운 엔트리를 할당받으면 안된다. 따라서 FDT를 매핑하기 위해서는 섹션 매핑을 하거나, 기존의 PTE를 공유하도록 만들어야 한다. 이런 이유로 2번 항목이 존재했다.
 
### 2. 2MB Boundary rule
&nbsp; **3번 항목**이 존재한 이유는, 4K paging일때 한 개의 섹션 매핑으로 FDT를 전부 매핑하도록 하기 위함이였다. 하지만 커밋이 된 이후, 이 항목은 사려졌는데, 코드를 살펴보면서 이 제약을 어떻게 없앴는지 알아보자.


![enter image description here](https://github.com/YWHyuk/YWHyuk.github.io/blob/master/img/Fixmap.png?raw=true)
&nbsp; 위 코드는 Fixmap 영역의 변동사항을 보여준다. 자세히 살펴보면, FIX_FDT_SIZE는 2MB가 아니라 4MB로 설정되어 있음을 볼 수 있다. 이는 FDT의 물리 주소가 2MB 경계선에 걸칠 수 있기 때문이다. 
## fixmap_remap_fdt 요약
<br>
&nbsp; FDT는 8Byte Align되어 있기에, 블록 매핑을 위해 2MB로 Round Down한 주소를 매핑하더라도, Magic과 Size(FDT의 처음 8바이트)를 접근할 수 있게 된다.

&nbsp; FDT를 사이즈를 읽은 다음 추가로 블록 매핑이 필요하면 나머지 부분을 매핑을 한다. 이 과정이 끝나면 FDT의 물리 주소는 Fixmap의 FIX_FDT로 매핑된다.
