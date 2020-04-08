---
layout: post
title:  "HacCTF bof pie write-up"
date:   2020-01-05 19:45:55
image:  hackctfbofpie.PNG
tags:   [HackCTF]
categories: [Write-up]
---

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20bof_pie/Untitled.png)

이 문제 역시 NX 비트와 PIE가 걸려있어 모든 영역의 쓰기 권한이 없고,  바이너리 주소가 상대적인 주소로 랜덤하게 매핑시키는 기법이다.

**ASLR과 비슷하게 생각하되, 차이점은 PIE는 Code(Text)영역을 포함한 모든 영역(Data, Stack, Heap, Libc)을 랜덤하게 매핑시킨다는 것이다.**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20bof_pie/Untitled%201.png)

프로그램을 실행시키면 다음과 같이 나온다

코드를 한번 봐보쟈

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20bof_pie/Untitled%202.png)

메인문에는 별게 없고 welcome() 함수가 있다

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20bof_pie/Untitled%203.png)

welcome 함수를 보면 "hello Do you know j0n9hyun?" 이라는 문자열 출력과 함께 welcome 주소를 출력하게 한다.

프로그램을 여러번 실행시키면 welcome의 주소는 pie에 의해서 변경된다.  8번 문제 처럼 오프셋을 이용하여 해결해야한다.

### 2. 접근방법

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20bof_pie/Untitled%204.png)

                         welcome 함수 오프셋

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20bof_pie/Untitled%205.png)

                           j0n9hyn 오프셋

두 함수의 오프셋은 다음과 같다

pie 에 의해서 함수의 base 주소가 계속 달라지지만 두 함수의 위치의 차는 그대로 고정이다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20bof_pie/Untitled%206.png)

이런식으로 메모리에 함수가 올라가있다.

base 주소가 계속 변경되지만 오프셋을 그대로 유지되므로 이를 이용하면 된다.

Welcome 함수의 offset이 0x909이고 J0n9hyn offset이 0x890이므로 두 차이는 0x79이다

그렇다면 welcome 함수 - 0x79 이 바로 j0n9hyun의 주소이다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20bof_pie/Untitled%207.png)

gdb로 확인해보면 생각한 대로 메모리가 구성되어 있다는 것을 알 수 있다

### 3. 풀이

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20bof_pie/Untitled%208.png)

1. J0n9hyun 까지 입력을 받고 welcome 함수의 주소를 저장한다
2. 해당 welcome 주소에서 0x79 오프셋위치를 저장한다
3. ret까지의 공간이 22바이트이므로 해당 페이로드를 작성해준다

### 4. 몰랐던 개념

pie 보호기법이 가지는 의미와 이에 해당하는 메모리 변화를 정확하게 알아야 할것같다