---
layout: post
title:  "Pwnable.xyz Sub write-up"
date:   2020-01-13 19:45:55
image:  pwnable_xyz_sub.PNG
tags:   [Pwnable_xyz]
categories: [Write-up]
---

### 1.  문제

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20sub/Untitled.png)

이 문제 역시 모든 보호기법이 다걸려있다

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20sub/Untitled%201.png)

프로그램을 실행시키면 숫자를 입력받는 공간이 나오고 숫자 크기에 따라서 Sowwy가 나오고 안나온다

코드를 봐보쟈

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20sub/Untitled%202.png)

int형 변수 v4, v5에 %u로 입력을 받고 두수의 크기가 4918 보다 작으면 조건문 안으로 들어온뒤, 

v4-v5 값이 4919이면 flag 를 출력해주는 것 같다

<br><br>

### 2. 접근방법

그렇다면 v4와 v5 값을 조정해서 v4-v5를 4919로 만들어주면 될 것이다. 하지만 v4와 v5는 입력할 수 있는 최대의 크기는 4918이다. 처음에 정수 오버플로우를 이용한 문제인 거같아서 삽질을 했지만 생각해보면 매우 쉬운 문제이다

처음에 선언한 변수의 타입을 보면 int 즉, signed int 형이다. 음수가 표현이 가능한 타입인 것이다.

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20sub/Untitled%203.png)

<br>

다음을 한번 봐보자

- int 변수 2개, unsigned int  변수 2개  에 각각 같은 음수를 집어넣었다
- 3번째 출력 문은 int 변수를 %u로 출력한 것이고
- 4번째 출력문은 unsinged int 변수를 %d로 출력한 것이다
- 6번째 출력문은 unsgined int 변수를 %d로 출력한 것이다

이로서 알수 있는 것은 다음과 같다

<br>

1. int든 unsigned int 든 사용자가 음수를 입력한다면 2의 보수 형태로 음수값이 표현되어 스택에 저장된다
2. 출력형태가 %d인지 %u인지 에 따라서 최상위 비트를 부호비트로 쓸 것인지, 아니면 그냥 전체를 값으로 볼지가 달라진다

그렇다면 int 형 변수에는 음수표현이 가능하므로 입력 값에 음수인 4019, -1 이렇게 두개 넣어도 가능할 것이다

<br><br>

### 3. 풀이

최종 익스 코드를 짤 필요도 없다

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20sub/Untitled%204.png)

<br><br>

### 4. 몰랐던 개념

- c언어의 출력타입 및 부호 값에 대한 지식이 헷갈렸는데 다시 정리할 수 있었다