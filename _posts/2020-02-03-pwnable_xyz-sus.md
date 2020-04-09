---
layout: post
title:  "Pwnable.xyz Sus write-up"
date:   2020-02-03 19:45:55
image:  pwnable_xyz_sus.PNG
tags:   [Pwnable_xyz]
categories: [Write-up]
---

### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20SUS/Untitled.png)

PIE가 안걸려있다. got overwrite가 가능할 것으로 보인다

<br>

**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20SUS/Untitled%201.png)

메뉴가 다음과 같이 나온다. 1번을 통해 user의 이름과 나이를 입력할수 있고, 2번을 통해 확인이 가능하다. 마지막으로 3번을 이용하여 이름과 나이를 수정 할 수 있다

<br>

**3) 코드 확인**

- **main**  

```c
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int v3; // eax

  setup(argc, argv, envp);
  puts("SUS - Single User Storage.");
  while ( 1 )
  {
    while ( 1 )
    {
      print_menu();
      printf("> ");
      v3 = read_int32();
      if ( v3 != 1 )
        break;
      create_user();
    }
    if ( v3 <= 1 )
      break;
    if ( v3 == 2 )
    {
      print_user();
    }
    else if ( v3 == 3 )
    {
      edit_usr();
    }
    else
    {
LABEL_13:
      puts("Invalid");
    }
  }
  if ( v3 )
    goto LABEL_13;
  return 0;
}
```
메인에는 별 문제 될것없다. 메뉴를 int형 변수 v3에 담아서 확인을 하는데, 그대로 4바이트 전체를 체크하기 때문이다. 

<br>

- **create_user**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20SUS/Untitled%202.png)

동적할당을 하여 해당 주소를 s에 넣는다. read 함수를 이용하여 최대 0x20 크기만큼 이름을 입력가능하다. 그리고 Age를 printf 하는데, read_int32() 함수로 입력을 받는다. 그다음 s가 담겨져 있는 주소를 전역변수 cur에 담고 끝나게 된다. 일단 아까 read_int32() 함수를 봐보자

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20SUS/Untitled%203.png)

buf 변수에 최대 0x20 크기만큼 입력을 하고 끝이난다. 나이는 힙 영역에 쓰는 것이 아니라는것을 알수 있다. 그렇다면 print_user에서 나이를 출력하는 부분이 어디를 참조하는지 확인해보자

<br>

- **print_user()**  

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20SUS/Untitled%204.png)

create_user가 실행되면 전역변수 cur에 s가 담겨져 있는 스택주소가 저장되어 있을 것이다.

print_user() 함수는 이 전역변수 cur에 담긴 데이터를 이용한다. cur에는 힙의 주소를 가리키는 포인터가 들어있다고 말했다. 그렇기 때문에 8라인의 printf에는 user 이름이 출력될 것이다.

이제 아까 나이를 어디부분에서 출력해주는지 확인해보자. cur에 담겨져 있는 포인터의+72위치에 있는 공간을 출력하기때문에 여기에 나이가 저장되어있는것을 알수 있다.

<br>

- **edit_user()**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20SUS/Untitled%205.png)

print_user 함수와 마찬가지로 cur 전역변수를 이용한다. 이를 통해 3개의 함수는 전역변수를 이용하여 데이터를 공유하는 것을 알 수 있다.

해당 함수를 통해 이름과 나이가 수정이 가능하다. cur에 0x20 만큼 다시 입력이 가능하다. 그다음 cur의 값(포인터)를 지역변수 v0에 저장한다음, read_int32() 함수의 리턴값을 +72위치에 4바이트로 저장한다.

<br><br>

### 2. 접근방법

---

이름과 나이 모두 0x20 크기만큼 입력가능하므로, 일단 아무렇게 넣어보자

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20SUS/Untitled%206.png)

처음에 create_user로 대충 만들고 2번으로 확인을 해보니 잘 나왔다. 다시 3번으로 수정을 하고 2번으로 다시 확인을 해보니 갑자기 세그폴트가 떳다. 왜 그런지 확인을 한번 해보자.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20SUS/Untitled%207.png)

print_user() 함수에서 printf에 bp가 걸려있는 상태이다.  rsi에 원래는 cur heap영역의 주소가 들어가 있어서, 힙에 저장되어있는 Name을 출력해야하는데, 0x3131313131.... 이게 들어가 있다. 0x31은 아스키 코드에서 확인해보면 문자 1의 16진수 값이다.

따라서 0x313131.. 의 영역에 있는 값을 printf하려고 하니까 해당 영역에는 접근권한이 없어 에러가 나게 됬던 것이다. 이 값은 우리가 아까 edit_user() 에서 1111111... 이렇게 입력했던 부분이 어딘가 덮혀진 것으로 추측이 가능하다.

따라서 이를 통해 edit_user()함수의 age를 입력하는 부분을 타겟으로 접근하면 될 것같다. 자세하게 어디서부터 데이터가 age를 통해 덮혀지는지  확인해보자

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20SUS/Untitled%208.png)

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20SUS/Untitled%209.png)

위 사진은 edit_user 함수에 bp를 걸고 age를 입력하기 전의 상태이다. read를 통해 age에 입력이 일어나는데, rax의 값을 rsi에 넣는다. rax는 0x..cc10이고, 힙의 주소는 0x..cc20 이다.

따라서 age를 입력할때 총 0x10(16개) 이상을 넘게 입력하면, 힙 주소를 가리키고 있는 부분을 변경할 수 있다. 

<br>

정리하자면, 1번을 통해 이름과 나이 아무값을 입력하고 3번 edit를 누른다. 이름은 아무거나 입력하고 나이를 입력할때  '1'*16 + printf의 got를 입력한다. 그다음 마지막으로 3번을 누르게 되면, cur에는 힙의 주소가 아닌, printf의 got가 들어가 해당 got의 부분에 입력을 받게된다.

여기에 win함수의 주소를 넣게된다면, printf의 got가 win함수로 변경되어 다음 printf 실행시 정상적인 실행이 아닌, win함수가 실행될 것이다

<br><br>

### 3. 풀이

---

최종 코드는 다음과 같다

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20SUS/Untitled%2010.png)

<br><br>

### 4. 몰랐던 개념

---

None