---
layout: post
title:  "Pwnable.xyz I33t write-up"
date:   2020-01-26 19:45:55
image:  pwnable_xyz_i33t.PNG
tags:   [Pwnable_xyz]
categories: [Write-up]
---


### 1.  문제

 

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20l33t%20ness/Untitled.png)

보호기법이 다걸려있다. 쉽지 않을것 같다

<br>

**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20l33t%20ness/Untitled%201.png)

뭔가 숫자를 입력받는다. 아무 숫자나 넣었는데 바로 끝나버렸다. 빠르게 코드를 확인해보자

<br>

**3) 코드 확인**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20l33t%20ness/Untitled%202.png)

조건문을 보면 round_1(), round_2(), round_3() 함수의 반환값들이 모두 참이면 win() 함수가 실행되어 플래그가 출력된다. 각 함수들을 확인해보자

<br>

- **round_1()**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20l33t%20ness/Untitled%203.png)

s 와  v4 변수에 각각 데이터를 입력한다. s는 char 형이지만 atoi 함수를 통해 int형으로 변경된다. strchr 함수를 이용하여 ' - ' 가 입력 문자에 들어있는지 확인하는데, 이는 음수의 입력을 막기위함으로 보인다.

하지만 인티저 오버플로우를 이용하여 음수 값 표현이 가능하다. 21번 라인에서 아까 입력한 값들을 검사하고, True 혹은 False를 리턴한다.

<br>

- **round_2()**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20l33t%20ness/Untitled%204.png)

v4, v5 변수에 signed int 형으로 입력을 받는데, 이 역시 특정 조건 검사 후 True 또는 False를 리턴함

<br>

- **round_3()**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20l33t%20ness/Untitled%205.png)

v2와 v3 그리고 v4에  일정 위치값으로 입력을 총 5개를 받는다. 사실 헥스레이만 보고 생각한 코드흐름이랑 직접 어셈을 확인하면서 디버깅한 흐름이랑 달랐다. 

17라인은 결국 v2를 기준으로 4바이트 오프셋에 위치한 값을 검사하는 로직이였고, 20라인도 v2기준 각 4바이트를 더한 값과 곱한값을 비교하는 로직이였다. 따라서 v2 변수에 'y'를 누르고, _DWORD[5] 로 타입캐스팅을 해주었다

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20l33t%20ness/Untitled%206.png)

더 보기 쉽게 바꼈다.

<br><br>

### 2. 접근방법

1) round_1() 함수는 결국, 1336 - (-1)  를 입력하면 된다. -1를 직접 입력하면 안되고, 인티져 오버플로우를 일으켜서 알맞게 변형시키면 된다

2) round_2() 함수 역시 인티져 오버플로우를 일으켜서 해당 조건에 맞게 세팅해주면 된다. v4는 1보다 크게, v5는 1337보다 크게 입력하고 두 값을 곱한 것이 오버플로우로 1337이 되게 하면 된다

3) round_3() 함수는 v2 기준으로 각 인덱스의 값이 점점 같거나 커져야 하고, 총4개의 값을 더한 값과 곱한 값이 같으면 된다.

<br><br>

### 3. 풀이

익스코드를 짤 필요도 없었다

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20l33t%20ness/Untitled%207.png)

<br><br>

### 4. 몰랐던 개념

- 모르는 개념은 없었지만, 리얼월드에서는 직접 변수 타입을 측정해서 구조체를 만들어서 변경해줘야한다고한다. 잘 습득하