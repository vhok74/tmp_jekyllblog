---
layout: post
title:  "Pwnable.xyz Add write-up"
date:   2020-01-14 19:45:55
image:  pwnable_xyz_add.PNG
tags:   [Pwnable_xyz]
categories: [Write-up]
---

### 1.  문제

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20add/Untitled.png)

이번 문제에는 PIE가 안걸려 있다

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20add/Untitled%201.png)

실행을 시켜보면 뭐.. 다음과 같다

아이다로 코드 흐름을 봐보자

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20add/Untitled%202.png)

- v4, v5, v6 int64 변수와, v7 배열을 하나 선언하고

- 각 변수에 입력을 받는다

- v7 배열의 v6 인덱스에 v4와v5 값을 더한것을 넣고 해당 부분을 출력해서 보여준다

처음에 아무 생각없이 33 55 66 이렇게 입력을 해보았다

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20add/Untitled%203.png)

33와 44를 더한것이 77인데 이것이 바로 출력되었다?

인덱스 값은 v7[55]일텐데.....

<br><br>

### 2. 접근방법

Out of Bound 문제이다. 인덱스를 설정한 값보다 큰 값을 넣을 수 있고 ret 주소까지 덮을수 있다. 해당 문제에서 win() 함수라는 플래그 출력 함수를 제공해줬으므로 ret 주소를 저 주소로 변경하면 될 것이다

<br>

그리고 PIE와 ASLR이 꺼져 있으므로  win 주소 그대로 사용 하면 될 것이다

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20add/Untitled%204.png)

현재 메모리 구조가 이렇게 구성되어 있다

따라서 v[13] = win() 주소 이렇게 주면 된다

- win 주소

    ![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20add/Untitled%205.png)

<br><br>

### 3. 풀이

- 첫번째 인자 v4 : win()주소의 10진수
- 두번째 인자 v5 : 0
- 세번째 인자 v6 : 13

여기까지 입력하면 ret주소는 변경되었고 scanf 탈출을 위해 문자열 아무거나 입력하면 된다

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20add/Untitled%206.png)

<br>

### 4. 몰랐던 개념

- scanf의 리턴값은 서식문자의 개수이다. 만약 scanf가 정상적으로 입력을 받지않을 경우 0을 반환한다

        a=1, a1=2, a2=3, b1="aa"
        
        1. scanf("%d %d %d",a, a1, a2)     ->   리턴값 3
        2. scanf("%d", a)                  ->   리턴값 1
        3. scanf("%d", b1)                 ->   리턴값 0 

<br>

- **Out-of-Bound**

    str[index] = 5; 이렇게 배열의 항목을 지정할 떄 상수가 아닌 변수를 사용한다면 컴파일러는 인덱스 값을 예측할 수 없기 때문에 오류처리가 불가능하다

    따라서 그대로 기계어로 번역할 것이고 인덱스 값이 배열의 크기를 넘어갈수 있고 크기가 5인배열이, str[8]=5 라고 그대로 들어가게 된다. 이런 인덱스에 대한 유효성을 개발자가 체크하는것이 맞다고 한다