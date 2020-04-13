---
layout: post
title:  "Pwnable.xyz Strcat write-up"
date:   2020-02-03 19:45:55
image:  pwnable_xyz_strcat.PNG
tags:   [Pwnable_xyz]
categories: [Write-up]
---


### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20strcat/Untitled.png)

NX 비트 말곤 안걸려있다. PIE가 안걸려있기 때문에 got overwrite가 가능할 것으로 보인다

<br>

**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20strcat/Untitled%201.png)


문제 이름처럼 이름과 나이를 입력하고 이를 수정 및 확인이 가능하다. 또한 1번 메뉴를 이용하여 이름을 이어 붙일 수 있다. strcat 함수의 기능을 수행하는 것으로 보인다

<br>

**3) 코드 확인**

    int __cdecl __noreturn main(int argc, const char **argv, const char **envp)
    {
      int v3; // eax
      int v4; // ebx
      int v5; // ebx
      size_t v6; // rax
    
      setup(argc, argv, envp);
      puts("My strcat");
      maxlen = 0x80;
      printf("Name: ");
      maxlen -= readline(name, 0x80);
      desc = (char *)malloc(0x20uLL);
      printf("Desc: ");
      readline(desc, 0x20);
      while ( 1 )
      {
        print_menu();
        printf("> ");
        v3 = read_int32();
        switch ( v3 )
        {
          case 2:
            printf("Desc: ");
            readline(desc, 0x20);
            break;
          case 3:
            printf(name);
            printf(desc);
            putchar(10);
            break;
          case 1:
            printf("Name: ");
            v4 = maxlen;
            v5 = v4 - strlen(name);
            v6 = strlen(name);
            maxlen -= readline(&name[v6], v5);
            break;
          default:
            puts("Invalid");
            break;
        }
      }
    }

메인 문은 readline() 함수를 이용하여 이름과 나이를 입력받는다. 그리고 read_int32() 함수를 이용하여 메뉴를 입력한다. 하지만 read_int32() 함수에 들어가보면 이 함수 역시 readline() 함수를 이용하는 것을 볼 수 있다. 한번 직접 봐보자

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20strcat/Untitled%202.png)

0x20 크기 만큼 동적할당 받은 주소에 메뉴를 입력하고 이를 atoi 함수를 이용하여 정수로 변환해주고 해당 값을 반환하는 함수이다. 해당 함수는 [pwnable.xyz](http://pwnable.xyz) 문제에서 자주 볼 수 있는 입력 함수이다. 이번에는 readline() 함수를 한번 봐보자

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20strcat/Untitled%203.png)


read 함수를 이용하여 a1의 주소에 a2 사이즈 만큼 입력을 받는다. realine() 함수는 해당 문제에서 입력받는 모든 곳에서 이용한다. 전역변수로 저장되어있는 7번째 라인은 입력한 하위 한바이트를 0으로 만드는 로직이다.

메뉴를 입력받는 read_int32() 함수에서는 리턴값이 필요없지만 switch case 문에서는 해당 readline()함수를 직접 사용하기 때문에 리턴값에 의미가 있다. 어쨋든 8라인은 현재 name 변수에 저장되어 있는 문자열의 사이즈에서 -1 뺀 값을 반환한다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20strcat/Untitled%204.png)


위에서 말한 readline() 함수의 반환값이 필요로 하는 부분은 case 1인 이름을 이어붙이는 역할을 수행하는 로직이다. 초기 0x80 사이즈의 maxlen을 기준으로 계속 입력한 이름의 크기만큼 빼주는 로직을 확인 할 수 있다.

또한 13라인에서 동적할당한 주소를 desc변수에 저장하는데 이의 위치는 다음에서 확인 가능하다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20strcat/Untitled%205.png)


name[128] 배열 바로 다음에 desc이 나온다. 즉 name+0x80 ~ 0x88 위치에 desc 주소가 들어가 있고, desc를 이용하는 부분에서 해당 주소를 참조할 것이다. 그렇다면 OOB를 이용하여 desc 변수의 값을 덮으면 될 것이다

<br><br>

### 2. 접근방법

---

name은 maxlen의 크기에 따라서 입력이 들어간다. 초기 maxlen은 0x80, 즉 128 이기때문에 128 크기를 넘어서 입력할 수 없고, desc를 덮을 수 없다. 정상적인 방법으로는 말이다.

초기에 생각한 방법으로는 1번 메뉴를 이용하여 이름을 계속 이어붙이면 maxlen은 점차 작아질 것이고,

    maxlen -= readline(&name[v6], v5);

이 부분이 계속 수행된다면 maxlen= maxlen - readline() 이 음수값이 들어 갈수 있다. 따라서 해당 방법을 수행하였지만,  readline 함수의 두번째 인자의 자료형은 singed int 이기 때문에 0xfff... 이렇게 바뀐 maxlen을 음수로 판단하여 입력을 받지 않고 넘어가게된다.

그렇다면, maxlen 을 음수로 만들지 않고 0x88 크기로 확대시킬수 있는 방법을 생각해야한다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20strcat/Untitled%206.png)


readline() 함수를 다시 봐보자. 만약 이름에 아무것도 입력하지 않는다면 strlen은 0을 반환하고 이 값을 v2에 들어갈 것이다. 그리고 return v2-1 에서 0-1 이 되므로 결국 -1을 반환하게 된다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20strcat/Untitled%207.png)


0x400a54 라인을 봐보자. 이는 처음 이름을 입력받고 maxlen=maxlen - realine()이 진행되는 부분이다. edx에는 0xffffffff이 들어가있고, eax에는 초기 사이즈인 0x80이 들어가 있다. 여기서 readline 함수의 반환 값은 unsigned 형태이므로 0xffffffff은 4294967295 값이다.

컴퓨터에서 음수는 2의 보수 형태로 표현하므로 다음과 같이 계산될 것이다.

    0x80 - 0xffffffff = 0x80 + 0xffffffff의 2의 보수

0xffffffff의 2의 보수를 취하면 1이 된다. 따라서 Integer underflow가 발생하여 0x80+1 = 0x81로 연산의 결과가 나오게 된다. 

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20strcat/Untitled%208.png)


sub eax, edx의 결과가 0x81이 되는 것을 확인 할 수 있다. 해당 취약점을 이용하여 0x88까지 maxlen을 증가시키면 desc에 저장된 값을 원하는 값으로 수정할 수 있다. 따라서 0x88위치에 printf의 got 주소를 넣고 2번 메뉴를 이용하여 printf_got값을 win함수의 주소로 변경시키면 된다

여기서 하나더 해줘야 하는 것이 있다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20strcat/Untitled%209.png)


위 사진은 readline() 함수에서 마지막 한바이트를 0으로 채우는 로직이다. 따라서 desc의 위치에 printf_got인 0x602040을 넣게 되면 little endian 형식으로 \x40\x20\x60으로 들어가고, \x60 부분이 널값으로 변경된다. 

따라서 아무값 한바이트를 더 줘서 got가 변경되지 않게 해줘야 한다.

<br><br>

### 3. 풀이

---

최종 익스 코드는 다음과 같다

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20strcat/Untitled%2010.png)


<br><br>

### 4. 몰랐던 개념

---

생각좀 하자.

<br><br>

### 5. 다른 문제 풀이

---

해당 문제를 풀고 다른 롸업을 봤는데 fsb로 문제를 푼 사람이 있었다. 코드를 자세히 보니 2번 메뉴에서 이름과 나이를 서식문자 없이 출력하는 printf가 있었다. 왜 이부분을 보지 못했는지,,,

(못본게 다행인거같기도하다 왜냐하면 뭔가 이상하긴한거같기도..)

어쨋든 fsb로 해당 문제를 풀려면 double stage fsb라는 기법을 이용해야한다. 문자열을 저장하는 버퍼가 지역변수가 아닌 전역변수에 저장되어 있기 때문이다.

fspoo 문제를 풀때 공부했기 때문에 쉽게 접근할줄 알았지만, 아니였다.. 왜냐하면 printf(%p) 이렇게 주면 현재 rsp 다음 값이 leak되야하는데 어딘지 모르는 주소가 leak 되었다.(내가 모르는 거일 수도)

어쨋든 Name에 aaaaa 이렇게 5개 정도 주고, desc에 %6$p를 준뒤 3번 메뉴를 선택하게되면,  현재 rsp값이 leak이 된다.  현재 rsp가 가리키는 값은 rsp+0xf0의 주소이다. rsp+0xf0에는 1이 들어가 있다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20strcat/Untitled%2011.png)

따라서 2번 메뉴를 이용하여 Name에 "AAAAA" 이렇게 주고, Desc에 %(put_got주소값을 10진수로)c%6$lln 이렇게 주면 0x7fffffffdd70 주소에 printf_got가 들어가게 된다. 여기서 $ln 이렇게 주게 되면 8바이트 공간에 값을 삽입할수있다. 64bit에서는 이렇게 해야한다고 한다. 

이렇게 put_got주소를 넣고, 한번더 2번 메뉴를 이용하여 win함수의 주소를 아까 put_got 주소를 넣은 위치에 삽입하면 된다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20strcat/Untitled%2012.png)

요렇게 0x7ff..fde20의 주소에 들어있던 1이 0x602028로 바뀐걸 확인 가능하다. 저 위치는 %36$ln으로 접근하면 된다. 왜냐하면 0x7fffffde20이 %6$ln 이기 때문이다.

따라서 2번 메뉴를 한번더 입력하여 %(win함수주소값10진수)c$ln 을 Desc에 입력하면 put_got가 다음과 같이 바뀐다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20strcat/Untitled%2013.png)

참고로 초반에 이상하다고 하는 부분에 대해서 이유를 알았다. 64bit에서는 함수호출 시 레지스터로 인자값을 이용하기 때문에 6개의 레지스터가 릭이 되고, 그다음 스택의 값이 릭이 된다고 한다. 아래 사이트에서 설명이 잘나와있다

[[Tip] Format String Bug 팁](https://holinder4s.tistory.com/29)