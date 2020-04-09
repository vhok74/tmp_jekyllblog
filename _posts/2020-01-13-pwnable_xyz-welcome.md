---
layout: post
title:  "Pwnable.xyz Welcome write-up"
date:   2020-01-13 19:45:55
image:  pwnable_xyz_welcome.PNG
tags:   [Pwnable_xyz]
categories: [Write-up]
---


### 1.  문제

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20welcome/Untitled.png)

1번 문제에 걸려있는 보호기법들은 다음과 같다. NX 비트가 걸려있기 때문에 실행권한도 없을 것이고, PIE도 걸려있어 GOT overrite도 불가능 할 것이다. 쩃든 프로그램을 한번 실행시켜보자

프로그램을 실행시키면 다음과 같은 문구가 뜨고 입력할 수 있는 공간이 뜬다

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20welcome/Untitled%201.png)

뭔지는 모르지만 어떤 주소를 하나 leak 해주고, 입력할 메시지 크기를 입력받는다.

그다음 메시지를 입력하게되고, 입력한 문구가 출력되면서 프로그램이 종료된다

코드를 한번 봐보자

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20welcome/Untitled%202.png)

문제를 보니 힙 문제인것 같다.

<br>

- 14라인을 보면 0x40000 크기만큼 동적할당을 한 뒤 v3변수에 저장을 한다
- 그다음 16라인에서 v3 주소를 알려준다. 뭔가 이 부분을 이용해야 하는것 같다
- v5에 20라인에서 입력한 사이즈만큼 동적할당을 한다
- 22라인에서 아까 입력한 사이즈만큼 v5변수에 입력한다
- v5의 마지막 인덱스에 0을 넣는다
- 26라인에서 동적할당한 v3 값이 0인지 아닌지 판단한다. 여기서 0이여야지 system 함수가 실행된다

<br>

대충 문제 흐름은 파악했다. 하지만 하나더 문제가 있다. gdb로 까보면 평소에 사용했던 명령어들이 실행되지 않는다

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20welcome/Untitled%203.png)

<br>

해당 프로그램을 컴파일 할때 디버깅 심볼등을 추가해주지 않았고, stripped되었기때문에 그렇다

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20welcome/Untitled%204.png)

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20welcome/Untitled%205.png)

- file 명령어를 이용하여 해당 파일의 정보를 확인하였는데 stripped 된 것을 볼 수 있다
- 오른쪽 사진은 임의로 만든 코드를 컴파일하여 file 명령어로 똑같이 확인해보았는데 stripped 되어 있지 않은 것을 볼 수 있다

<br>

그렇다면 임의로 main 함수의 주소를 직접 찾고, 해당 주소에 bp를 걸면서 디버깅을 진행해야 할 것이다

<br><br>

### 2. 접근방법

우선 메인함수의 주소를 찾아 보자

- **사용 명령어**
    1. (gdb) start
    2. (gdb) vmmap

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20welcome/Untitled%206.png)

저기가 base 주소이므로 저 주소에 아이다에서 확인한 메인함수의 오프셋을 더해주면 된다

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20welcome/Untitled%207.png)

main 함수 주소 : 0x0000555555554000 + 0x920 

<br>

따라서 저 주소에 bp를 걸고 디버깅을 진행하면 된다.이제 모든 준비가 다 끝났다

<br>

현재 메모리 구조를 보면 대충 다음과 같다

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20welcome/Untitled%208.png)

- v3주소를 leak해줘서 알려주고
- v5 변수를 원하는 크기 만큼 할당 및 데이터 입력이 가능하다
- 그리고 v5[ size-1 ] = 0 이 진행이 된다

생각보다 간단하다.

v5 변수에 동적할당을 할때 만약 size를 엄청 크게 입력하게 된다면 정상적으로 할당이 되지 않고 NULL을 반환하게 된다. man malloc을 이용하여 확인해보자 

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20welcome/Untitled%209.png)

malloc 에러시 NULL을 리턴한다고 한다. 그렇다면 언제 malloc이 실패하는 경우 일까?

바로 메모리에서 접근할 수 있는 용량 제한을 넘어가면 실패한다고 한다. 즉 사이즈를 이빠이 주고 할당해줘 라고 하면 실패할 수 있다는 것이다!

<br>

### 그렇다면!!

v5에 malloc 실패를 유도해서 NULL을 반환, 즉 v5 주소를 0으로 만들고  
- v[size - 1] = 0

이 부분을 이용하면 된다. 해당 라인을 어셈블리어로 보면 다음과 같다

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20welcome/Untitled%2010.png)

- rbp : v5 주소
- rdx : 입력한 크기

<br>

만약 v5 주소를 0으로 만들게 되면

[ 0+입력한사이즈 - 1 ] 의 주소에 들어있는 값과 0을 비교하게 된다.

그렇다면 size에 leak 주소 +1 을 입력하게 된다면

[ 0 + leak주소+1 - 1 ] = leak 주소가 될 것이고, 저 주소에는 지금 

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20welcome/Untitled%2011.png)

1이 들어가 있는데 이거를 0x0으로 덮게 되므로 v3은 0으로 변경될 것이다

<br><br>

### 3. 풀이

최종 익스 코드는 다음과 같다

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20welcome/Untitled%2012.png)

<br>

### 4. 몰랐던 개념

1. **malloc 실패할 수 있는 경우**

    "malloc returns a void pointer to the allocated space, or NULL if there is insufficient memory available."

    할당된 메모리공간의 void pointer를 return 하거나, 메모리 공간이 부족하면 NULL을 return한다. (void pointer를 return 하므로 필요에 따라 casting을 해야 한다)

    내 우분투 환경에서 최대 얼마까지 할당이 가능할지 테스트를 해보았다  <br><br>

    ![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20welcome/Untitled%2013.png)

    대략 1G가 정도까지 할당이 가능하다

    문제에서 내가 입력한 size의 크기는 0x7F0BCDDD5010 이므로 충분히 실패하고도 남을 크기이다
    
<br>    

2. **binary stripped**

    프로그램이 컴파일 될 때 컴파일러 (이 경우 gcc)는 디버깅 심볼이라는 바이너리에 추가 정보를 추가한다.

     이 기호를 사용하면 프로그램을 쉽게 디버깅 할 수 있다. 제거 된 바이너리는 컴파일러에서 이러한 디버깅 기호를 버리고 그대로 프로그램으로 컴파일하도록 지시 하는 스트립 플래그로 컴파일되는 프로그램이다

     바이너리를 제거하면 디스크의 크기가 줄어들고 디버그 및 리버스 엔지니어링이 조금 더 어려워진다. 벗겨진 바이너리와 잘리지 않은 바이너리 사이의 크기 차이는 다음과 같다

    ![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20welcome/Untitled%2014.png)

    ![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20welcome/Untitled%2015.png)

    디버깅 심볼 삭제 및 -s 옵션으로 stripped 된 바이너리의 크기가 더 작아진것을 확인 가능하다<br><br>

3. **(gdb) start**

    이 명령어는 main()에 temporary bp를 걸고 실행을 시작한다. 일반적으로 이 명령어를 통해 EP로 이동하여 디버깅을 시작할 수 있다<br><br>

4. **main 함수의 실행 원리**

    main 함수가 실행되기까지의 과정들은 다음과 같다

    ![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20welcome/Untitled%2016.png)

<br>
해당 내용은 깊이가 있는 부분이므로, 이부분만 디테일하게 따로 정리를 할 예정이다