---
layout: post
title:  MATLAB Programming 공부 01일차
comments: true
categories: Matlab
tags : [Matalab, Programming]
---

&nbsp; 대학생때는 매트랩을 깔면 다른 게임을 위한 용량이 부족했기에 최대한 사용을 피해왔지만, 업무에서 매트랩을 사용해야 하기에 매트랩을 공부하기 마음 먹었다. 
&nbsp; 매트랩 프로그램 활용법이나 toolbox 활용법보다는 곧바로 Data type와 Flow Control와 같은 Basic Component를 소개하는 ***MATLAB® The Language of Technical Computing*** 을 바탕으로 공부한 내용을 기록하겠다.

# Data Structure

매트랩(Matrix + Laboratory)의 이름과 같이 매트랩의 가장 기본적인 단위는 ***matrix*** 이다. Matrix는 2차원 행렬을 의미하고, 다양한 타입의 Matrix가 선언될 수 있다(int 타입 Matrix, char 타입 Matrix, ...).  

### Create Matrix

가장 쉽게 Matrix를 생성하는 방법은 ***[ , ]*** 을 사용하는 것이다. 하나의 ***row*** 는 다음과 같이 만들 수 있다. 각 요소들은 콤마 또는 스페이스를 통해 구분된다.
	
	row = [ E1, E2, E3, E4 ]
	row = [ E1 E2 E3 E4 ] 

새로운 row를 시작하려면, 세미콜론을 두 row사이에 표기하면 된다.

	A = [ row1; row2]
	A = [ 12 62 93; 16 2 87; -4 17 -7 ] 
	----------------<result>----------------
	A   =   12 62 93 
	        16  2 87 
	        -4 17 -7 
또한 [Specialized Matrix Functions](https://kr.mathworks.com/help/matlab/matrices-and-arrays.html)를 사용하여 Matrix를 생성할 수도 있다.
