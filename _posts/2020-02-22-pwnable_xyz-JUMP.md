---
layout: post
title:  "pwnable.xyz J-U-M-P write-up"
date:   2020-02-22 23:40:55
image:  pwb_JUMP.PNG
tags:   [Pwnable_xyz]
categories: [Write-up]
---

### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/J_U_M_P/1.png)

스택 카나리를 제외하고 모든 기법이 다 걸려 있다

<br>

**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/J_U_M_P/2.png)

문제를 실행시키면 점프점프 거리면서 뭐라고 나온다. 3번 메뉴를 누르면 스택 주소처럼 보이는 주소가 출력이 된다. 해당 주소를 이용해서 뭐 어떻게 하는 것같다

<br>

**3) 코드 확인**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/J_U_M_P/3.png)

메인 문은 다음처럼 생겼다. 해당 바이너리에 스택 카나리 보호기법이 걸려있지는 않지만, 직접 구현하여 gen_canary() 함수를 통해 체크하는 부분이 보인다.

read_int8() 함수로 입력을 받고, 1을 입력하면 rax에 들어있는 값으로 jmp를 하게되고 2를 입력하면 xor 연산을 한다. 마지막으로 3번을 입력하면 어떤 주소값을 출력해준다. 내가 입력할 수 있는 공간은 read_int8() 함수 밖에 없다. 해당 함수를 살펴보자.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/J_U_M_P/4.png)

자세히보면 buf는 rbp-0x20 공간에 할당되는데 입력받는 사이즈는 최대 0x21바이트이다.  해당 부분이 의심스럽다

<br><br>

### 2. 접근방법

---

그렇다면 read_int8() 함수를 이용하여 rbp(main의 rbp값이 들어 있는)의 한바이트를 변경 가능하다. main의 rbp를 변경함으로서 얻을수 있는 것을 무엇일까?. 어셈블리 코드를 한번 살펴보자.

    0x0000000000000bf5 <+107>:   mov    rax,QWORD PTR [rbp-0x8]
    0x0000000000000bf9 <+111>:   jmp    rax

해당 부분은 메뉴 1번을 눌렀을 때 실행되는 어셈이다. rbp-0x8에 위치한 값을 rax에 넣고, rax로 jump를 하는데 rbp를 수정함으로써 mov rax, qword ptr [rbp-0x8] 라인을 이용하여 rax 값을 수정할 수 있다.

가령, 기존 rbp (0x1248)-0x8  라인이 있다고 가정해보자. 근데 rbp 값을 수정하여 0x1240으로 바꿨다면 rbp (0x1248)-0x8 를 통해 0x1240 의 주소에 들어있는 값을 가져오는 것이 아닌, rbp(0x1240)-0x8

의 결과인 0x1238 주소에 들어있는 값을 가져오게 된다.

아래 코드를 다시 봐보자

    0x0000000000000bf5 <+107>:   mov    rax,QWORD PTR [rbp-0x8]
    0x0000000000000bf9 <+111>:   jmp    rax

1번 메뉴를 이용하여 jmp rax가 실행된다. 그렇다면 rbp-0x8의 위치에 win 함수의 오프셋만 넣으면 된다.  원래 rbp-0x8의 위치에는 다음의 주소가 들어가있다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/J_U_M_P/5.png)

0xba0의 오프셋이 들어간다. 따라서 하위 한바이트를 0x77로 변경하면 된다. 그렇다면 어떻게 변경을 해야할지 생각을 해봐야한다.  

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/J_U_M_P/6.png)

위 그림을 자세보면 read_int8()로 메뉴를 입력받고 al 레지스터의 값을 rbp-0x11로 복사하고 그 값을 switch case문으로 비교하면서 메뉴에 맞는 작업을 수행한다.

만약 rbp값을 +9 만큼 수정한다면 rbp = rbp +9 로 변경될 것이고, 위 사진의 rbp-0x11은 rbp+9-0x11 = rbp -0x8로 수정되어 rbp-0x8 위치에  내가 입력한 값(한바이트)을 넣을 수 있다

그럼 이제 마지막으로 read_int8의 rbp의 값을 변경해보자. 해당 rbp는 main 함수 스택프레임의 rbp를 가리키고 있기때문에 main rbp의 값이라고 하자.  3번 메뉴를 통하여 스택의 임의 주소가 leak이 된다고 했다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/J_U_M_P/7.png)

leak된 주소에서 0xf8 크기 만큼 떨어진 곳에 rbp가 존재한다. 이 값을 이용하여 read_int8()함수에서  "A"*0x20+rbp(마지막한바이트+9) 를 입력하면 rbp의 한바이트가 현재 값에서 +9 만큼 증가된다.

그다음 반복물을 돌고 다시 메뉴를 입력하게 되면 입력한 값이 rbp-0x11 이 아닌 rbp+9-0x11 (= rbp -0x8) 에 들어간다. 따라서 이 부분에 0x77을 입력하고, 카나리 조건문을 통과하기 위해 rbp를 원상복구 시켜야 한다. 이게 무슨 말이냐면

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/J_U_M_P/8.png)

해당 코드는 메인문의 카나리 검사 로직인데 rbp가 변조가 된상태로 1번 메뉴를 실행하게 된다면 rbp-0x10을 가리키는 곳이 rbp+9-0x10이 되므로 조건에 맞지 않게 된다. 따라서 rbp를 다시 되돌려놔야 해당 조건을 만족하게 되고,  win 함수로 jmp를 할 수 있게 되는 것이다.

<br><br>

### 3. 풀이

---

최종 익스 코드는 다음과 같다

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/J_U_M_P/9.png)

<br><br>

### 4. 몰랐던 개념

---

생각좀 하자 제발.