---
layout: post
title:  "Pwnable.xyz Misalignment write-up"
date:   2020-01-15 19:45:55
image:  pwnable_xyz_mis.PNG
tags:   [Pwnable_xyz]
categories: [Write-up]
---

### 1.  문제

모든 보호기법이 다 걸려 있다

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20misalignment/Untitled.png)

<br>

빠르게 아이다로 코드를 봐보자

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20misalignment/Untitled%201.png)

add 문제와 비슷한 문제인듯 싶다. v6, v7, v8에 입력을 받고, v6+v7 값을 v5 배열의 특정 인덱스에 넣는다

- 인덱스가 사용자의 입력에 따라 변경이 가능하고, QWORD 사이즈 즉, 8바이트 만큼 캐스팅을 하므로 OOB가 가능할 것으로 보인다. 이를 이용하여 v5[7] 값을 조건문에 맞게 변경해주자

<br><br>

### 2. 접근방법

처음에 헷갈려서 계속 그리면서 해봤다...

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20misalignment/Untitled%202.png)

<br>

- **결론!!**

    char 배열의 하나의 인덱스에 들어갈 수 있는 크기는 1바이트이지만 8바이트 캐스팅을 하므로 OOB 가 가능.

    ![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20misalignment/Untitled%203.png)

    이렇게 값을 채워주면 리틀엔디안 형식으로 v7에 원하는 값이 들어갈 것이다

<br><br>

### 3. 풀이

최종 익스 코드는 다음과 같다

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20misalignment/Untitled%204.png)

<br><br>

### 4. 몰랐던 개념

- None