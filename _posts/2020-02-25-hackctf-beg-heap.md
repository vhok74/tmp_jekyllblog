---
layout: post
title:  "HacCTF beginner heap write-up"
date:   2020-02-25 19:45:55
image:  hackctf_beginner_heap.PNG
tags:   [Hackctf]
categories: [Write-up]
---

# [HackCTF] Beginner_Heap

Date: Feb 25, 2020
Tags: report


### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Beginner_Heap/Untitled.png)

PIE가 걸려있지 않기 때문에 got overwrite가 가능할 것이다

**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/(HackCTF%20Beginner_Heap/Untitled%201.png)

바이너리를 실행시키면 아무것도 뜨지 않고 바로 입력을 받는다. 총 2번 입력을 받고 바로 종료가 된다. 문제만 봐서는 어떤 동작을 하는 프로그램인 파악이 안가기 때문에 아이다로 코드를 한번 봐보자

**3) 코드 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Beginner_Heap/Untitled%202.png)

malloc을 총 4번 하고 fgets를 통해 입력한 데이터를 strcpy 함수를 이용하여 복사를 한다.

위 과정을 디버깅하여 취약점을 찾아내야 한다. 만약 취약점을 찾았다면, got를 덮을수 있는지봐야하고, 이게 가능하다면 제공된 함수 중 **sub_400826** 로 이동시키게 해야한다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Beginner_Heap/Untitled%203.png)

**sub_400826** 는 다음과 같다. 그냥 해당 함수를 호출하면 될것같지만 해보지않고서는 잘 모르니 일단 해당 함수를 호출할 수 있게끔 조져보자.

### 2. 접근방법

---

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Beginner_Heap/Untitled%204.png)

메모리 구조는 다음처럼 생겼다. 처음 v3에 0x10 크기 만큼 할당 받고, v3이 가리키는 힙영역에 1을 넣는다. 그다음 v3+1가 가리키는 곳에 0x8 사이즈 만큼 한번더 동적할당을 한다. 근데 자세히 보면 해당 영역은 v3에서 0x20 만큼 떨어져 있다.

그리고 fgets에서 큰 사이즈만큼 입력이 가능하다. 이 부분에서 취약점이 발생한다. 입력한 데이터를 v3+1이 가리키는 곳에 복사하는데 이는 v4와 거리가 얼마 안떨어져 있다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Beginner_Heap/Untitled%205.png)

만약 0x30 크기 만큼 입력을 하고 strcpy를 실행한다면 v4+1위치에 들어있는 값을 수정가능 할 것이다. 이 부분을 Exit함수의 got로 변경해보자.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Beginner_Heap/Untitled%206.png)

(중간에 롸업쓰다가 멈춰서 주소가 달라졌다. 참고하시길)

0x602058 주소에 Exit_got가 잘 들어가 있다.  fgets에 sub머시기 함수의 주소를 입력하면 strcpy를 통하여 got에 들어있는 값을 변경 가능하다

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Beginner_Heap/Untitled%207.png)

정상적으로 got가 sub_400826 주소로 변경되었다. 이제 메인에서 exit(0) 실행되면 sub_400826 함수가 실행될 것이다.

### 3. 풀이

---

최종 익스 코드는 다음과 같다

    from pwn import *
    context.log_level = "debug"
    #p = remote("ctf.j0n9hyun.xyz",3016)
    p=process("./beginner_heap.bin",aslr=False)
    gdb.attach(p,""" b* 0x4009ab """)
    payload = "A"*40+p64(0x0000000000601068) # exit got
    
    p.sendline(payload)
    p.sendline(p64(0x400826))
    
    
    p.interactive()

### 4. 몰랐던 개념

---

힙 관련 지식은 딱히 필요없었다.