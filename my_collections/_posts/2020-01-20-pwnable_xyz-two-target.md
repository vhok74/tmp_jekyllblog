---
layout: post
title:  "Pwnable.xyz Two target write-up"
date:   2020-01-20 19:45:55
image:  pwnable_xyz_twotarget.PNG
tags:   [Pwnable_xyz]
categories: [Write-up]
---


### 1.  문제

50점 마지막 문제이다. 해당 프로그램은 PIE가 걸려있지 않음으로, got overwrite가 가능하다

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20two%20targets/Untitled.png)

코드를 한번 봐보자

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20two%20targets/Untitled%201.png)

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20two%20targets/Untitled%202.png)

결국 auth() 함수의 리턴값이 true가 되야지 win() 함수가 실행될 것이다

auth 함수를 한번 봐보자

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20two%20targets/Untitled%203.png)

어쩌구저쩌구 복잡한 수학적인 계산을 진행한 뒤,

s1와 s2에 담긴 값들을 0x20 만큼 비교를 한다.

어찌됫던간에 0x20 크기 만큼의 문자열이 동일해야지만 **strncmp == 0** ;  이 구문이 참이 될 것이고, 그렇게 되면 메인함수에서 **if( auth() )** 조건을 만족하여 win 함수가 실행 될 것이다.

<br><br>

### 2. 접근방법

처음에는 저 auth 함수의 수학적 연산에 충족하는 값을 만드려고 했다. 하지만 계산을 하면 할 수록 뭔가 이게 아닌데.. 라는 느낌이 들었고, 스승님의 힌트를 통해 문제의 의도를 파악할 수 있었다.

**two targets Which one would you exploit?**

문제에서 이렇게 말한다.

two targets 이라는 문구는 타겟이 2개다 라는 말이라기보다, 서구적인 표현으로 한쪽이 실제 타겟이고 나머지 타겟은 미끼로 쓰인다는 뜻이라고 한다. 

보통 소설에서 **two targets, double targets** 이라는 문구가 이러한 의미로 쓰인다.

그렇다면 뻔히 보이는 auth 함수의 수학적 연산을 통한 접근은 미끼고, 다른 식의 접근이 가능 할 것이다.

<br>

1. **첫번째 시도**
    - 무작정 값 때려넣어보기

        ![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20two%20targets/Untitled%204.png)

        메인함수에서 입력에 사용하는 변수는 총 3개이다.

        메뉴에서 1번을 누르면 s
        메뉴에서 2번을 누르면 v5
        메뉴에서 3번을 누르면 v6

        들의 변수에 값이 각각 들어간다

        s 와 v5 는 0x20 만큼 떨어져 있고
        v5와 v6는 0x10 만큼 떨어져 있다
        char s 는 다음 위치에 존재한다

        ![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20two%20targets/Untitled%205.png)

        v5는 다음 위치에 존재한다<br><br>

        ![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20two%20targets/Untitled%206.png)

        v6는 다음 위치에 존재한다.<br><br>

        ![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20two%20targets/Untitled%207.png)

        ???????????????????????????<br><br>

        원래대로라면 v5와 0x10 만큼의 위치인 0x7fffffffdc50 위치에 v6 변수가 존재해야하는데 현재 0으로 되어있다. 뭔가 이상하다

        해당 부분은 다음과 같이 되어있다

        ![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20two%20targets/Untitled%208.png)<br><br>

        int64 v6 변수에 scanf로 값을 넣으려면 v6 변수의 주소값을 넣어줘야 하기때문에 &v6 이렇게 되야하는데, 현재 v6 값 자체가 인자로 들어간다

        그렇다면 v5로 v6 영역을 덮고, 원하는 동작을 일으키게 하면 될 것이다<br><br>

2. **두번째 시도**
    - 그렇다면 v6 영역을 어떠한 값으로 덮고, 어떠한 동작이 일어나게 해야할까?

    의외로 간단했다.

    해당 문제는 PIE가 안걸려있기 때문에 got overwrite가 가능하다고 했다. 그렇다면 특정 함수의 got 으로 v6를 덮고, 그 got에 win 함수의 주소를 넣어 주면 끝이다<br><br>

3. **세번째 시도** 
    - 덮을 got를 선택하자

    1. **printf**

        ![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20two%20targets/Untitled%209.png)

        - printf의 got는 0x603038이고, 이곳에 다음과 같은 값이 들어가있다.
        - 초반에 이미 printf가 실행되었기 때문에 got table에는 이미 printf 함수의 실주소가 들어가 있다

        - 현재 메뉴 3번은 %d로 입력을 받기 때문에 최대 4바이트 입력이 가능하다
        - 따라서 4바이트 씩 두번에 걸쳐서  got 값을 변경해줘야 한다<br><br>

        ![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20two%20targets/Untitled%2010.png)

        - 하지만 한번 got가 이렇게 하위 4바이트가 변경되고 다시 메인으로 돌아와 처음 printf 호출시 이상한 got를 호출하게 되므로 에러가 난다.
        - 따라서 printf got의 값을 변경시키는 것은 불가능이다<br><br>

    2. **puts**
        - puts 함수는 사용자가 임의적으로 이상한 값을 넣어서 Invalid 가 출력되지 않는 이상 사용되지 않는다
        - puts의 got주소는 0x603020이다

            ![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20two%20targets/Untitled%2011.png)<br><br>

        - 하지만 scanf의 특성상 space를 만나면 거기까지만 입력을 받는다

        - v6을 덮기위해서 v5에 **<아무값*16 + 0x603020>** 을 입력해야하는데, 이는 %s로 받으므로 문자열 형태를 입력해야한다

            0x20 :  space  
            0x30 :  " 0 "  
            0x60 :  "  `  "

        - 따라서 마지막 4바이트를 이렇게 입력해줘야하는데 space 를 기준으로 짤리므로 원하는 값을 v6에 넣을 수 없다
        - 결론적으로 puts도 불가능이다<br><br>

      3.  **strncmp**

    - strncmp의 got는 0x603018이다

        ![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20two%20targets/Untitled%2012.png)

        0x18 :  cancel → ^X  
        0x30 :  " 0 "  
        0x60 :  "  `  "  

    - 해당 got는 문제가 없어 보인다
    - 따라서
        1. 메뉴1번을 이용하여 v5 = "a"*16 + ^X+'0'+`
        2. 메뉴 2번을 이용하여  %d이므로 0x40099c 를 10진수 형태로 입력해준다
        3. 그다음 4번을 입력하여 auth함수가 실행되고 마지막 strncmp가 실행되도록 한다

<br><br>

### 3. 풀이

이 문제 역시 익스코드 대신 직접 계산한 값들을 넣어봤다

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20two%20targets/Untitled%2013.png)

<br><br>

### 4. 몰랐던 개념

- 몰랐던 개념은 딱히 없었지만 시야가 아직 좁다.
- pwntools가 간편한 도구이긴 하지만 이렇게 직접 gdb로 값들을 확인하면서 푸는게 지금은 도움이 더 되는것 같다.
- 열심히 하자