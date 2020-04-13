---
layout: post
title:  "Pwnable.xyz Xor write-up"
date:   2020-01-19 19:45:55
image:  pwnable_xyz_xor.PNG
tags:   [Pwnable_xyz]
categories: [Write-up]
---

### 1.  문제

해당 문제는 카나리 제외하고 다 걸려있다

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20xor/Untitled.png)

<br>

코드를 한번 봐보자

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20xor/Untitled%201.png)

<br>

코드흐름은 간단하다

1. while문으로 계속 입력을 받는다
2. v4, v5, v6 총 3개의 변수에 각각 입력을 받는다
3. 0을 입력하면 안되고, 또한 v6의 값은 9보다 커야한다
4. result 변수의 v6 인덱스의 위치에 v5와 v4를 xor 한 값을 집어 넣는다

 각 변수들은 int64, 즉 singed int64 (8바이트) 이므로 음수값도 삽입 가능하다

따라서 OOB가 가능할 것이다. 또한 vmmap으로 다음과 같은 특이점을 알 수 있다

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20xor/Untitled%202.png)

0x55..54000 부터 0x55..55000 사이의 주소에 모든 권한이 다있다. (rwx)

이 영역안에 .text 영역이 존재하는데, 그말인 즉슨 

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20xor/Untitled%203.png)

.text 영역, 즉 코드영역을 수정이 가능하다는 소리이다

<br><br>

### 2. 접근방법

코드영역을 변경 할 수 있다는 뜻은 코드를 변경하여 실행시킬 수 있다는 소리이다. 따라서 OOB를 이용하여 특정 코드의 변경도 가능하다. 이를 이용해서 win함수를 호출시키는 것을 방향으로 잡으면 된다

그렇다면, 어느 부분을 변경시켜 어떻게 win 함수를 호출시키면 될까?

우리가 현재 입력할 수 있는 공간은 .bss 영역에 존재하는 result 변수이다. 우선 위치를 한번 확인해보자

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20xor/Untitled%204.png)

result는 프로그램의 base 주소 + 0x202200 만큼의 위치에 존재한다. 코드영역의 위치를 한번 확인해보자

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20xor/Untitled%205.png)

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20xor/Untitled%206.png)

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20xor/Untitled%207.png)

메인함수의 어셈블리가 이렇게 되어있다

<br>

이 중 call ??? 부분을 call win으로 변경하면 될 것이다. 그렇다면 0xac8의 **call exit** 을 타겟으로 잡아보자

<br>

- 필요사항

    1. **result 변수와 call exit  즉 0x202200 부터 0xac8 사이의 오프셋을 구해보자**
        - call exit 까지의 오프셋 : 0x202200 - 0xac8 = 0x201738

        그렇다면 v6 변수 값과 오프셋을 이용하여 0xac8 위치를 가리키에 해야한다
        즉, result[ v6 ] 이 위치가 0xac8이 되게끔 해야한다 (물론 0xac8은 상대주소)

        여기서  간과하면 안되는 것이 있다. 아이다에서 헥스레이로 보면 그냥 *result[v6]* 이지만, 어셈으로 보면 *lea rdx, ds:0[rax*8]*  즉, v6 값*8 만큼이 들어간다

        정리를 해보면 결국

        *result + v6*8 = v5^v4* 

        이렇게 들어간다. 이는 아마도 8바이트 얼라인 때문에 그런것 같다. 그렇다면 v6에 들어가야 하는 값은

        - 0x202200 + 8*v6 = 0xac8
        - **v6 = -262887(10진수)**<br><br>

    2. **0xac8 주소에 들어가야 하는 값**

        현재 0xac8 위치에는 다음의 8바이트 값이 들어가 있다

        ![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20xor/Untitled%208.png)

        - **"call exit"**는 총 5바이트로 표시되는데
        - **opcode E8** : 0x58
        - **0xfffffd63** : 특정 기준점에서 상대적인 offset
            여기서 말하는 기준은 해당 call 명령어가 있는 다음 주소를 기준으로 한다

        즉, 0xac8 다음 0xacd를 기준으로 0xfffffd63 만큼 떨어진 곳을 호출한다는 의미이다. singed int 기준으로 0xfffffd63은 669(0x29d) 이므로 0xacd - 0x29d = 0x830 에 exit 함수의 plt 주소가 들어가 있을 것이다

        ![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20xor/Untitled%209.png)

        그렇다면 -669 가 아닌 win 함수까지의 오프셋을 구해서 조지면 된다. win 함수는 0xa21 이므로

        - 0xacd - ? = 0xa21

        따라서 ?는 172 이고

        - 0x100000000 - x = 172
        - x= 0xFFFFFF54 이다

        그렇다면 0xFFFFFF54e8 를 v5 ^ v4 값으로 맞춰주면 될 것이다

        xor 을 두번하면 자기 자신이므로 0xFFFFFF54e8  ^ 1  한 값을 v5로, v4는 1로 하면

        v4 ^ v5  는 결국

        - **v4(1)**   ^  **v5(0xffffff54e8^1)** 가  되어 0xffffff54e8이 될 것이다.<br><br>

        **정리하면** 

        1.  v4 = 1
        2.  v5 = 0xffffff54e9 (1099511583977)
        3.  v6  =  -262887

        를 입력하고 다음 while 문에서 빠져나와 exit이 실행되게끔 하면 된다

<br><br>

### 3. 풀이

익스 코드 대신 직접 값을 입력해보자

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20xor/Untitled%2010.png)

<br><br>

### 4. 몰랐던 개념

- **Relative call**
    - Relative call (opcode)은 **0xE8** 로 표현된다
    - 0xE8 뒤 4바이트는 call instruction 다음 주소를 기준으로 오프셋의 크기가 담긴다
    - 오프셋은 signed int 형 기준으로 계산되는데, 이는 다음과 같다

        **스승 :** 

        e8은 일단 32bit 아키텍쳐에서도 쓰이던게 그대로 쓰이는 거라 그런것도 있고요 64bit offset을 지원해줄 정도로 커다란 relative call이 필요한 바이너리가 있을지도 잘 모르겄네요

        찾아보면 int64 offset을 지원하는 relative call이 있을 수도 있는데 전 실제 뭐 분석하면서 본적은 없어요!

- 결론, 알아야 하는 것은 더 많고 더 열심히 해야겠다.