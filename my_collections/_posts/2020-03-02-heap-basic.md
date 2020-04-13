---
layout: post
title:  "Heap 기초1(glibc 2.23)"
date:   2020-03-02 19:45:55
image:  heap1.PNG
tags:   [Heap,Stduy]
categories: [Computer]
---
**목차**  
[1. 데이터를 저장하는 여러 방법들](#1-데이터를-저장하는-여러-방법들)  
&nbsp;&nbsp;[1.1 전역변수로 할당된 데이터](#11-전역변수로-할당된-데이터)  
&nbsp;&nbsp;[1.2 스택에 할당된 데이터](#12-스택에-할당된-데이터)   
&nbsp;&nbsp;[1.3 동적으로 힙에 할당된 데이터](#13-동적으로-힙에-할당된-데이터)  
[2. Heap 기초](#2-Heap-기초)  
&nbsp;&nbsp;[2.1 Dynamic memory allocator](#21-dynamic-memory-allocator)   
&nbsp;&nbsp;[2.2 Glibc malloc(2.23기준)](#22-glibc-malloc-223)      
&nbsp;&nbsp;&nbsp;&nbsp;[2.2.1 main thread에서 malloc 호출 이전](#221-main-thread에서-malloc-호출-이전)  
&nbsp;&nbsp;&nbsp;&nbsp;[2.2.2 main_thread에서 malloc 호출 이후, free 호출 이전](#222-main-thread에서-malloc-호출-이후,-free-호출-이전)    
&nbsp;&nbsp;&nbsp;&nbsp;[2.2.3 main thread에서 free 호출 이후](#223-main-thread에서-free-호출-이후)   
&nbsp;&nbsp;&nbsp;&nbsp;[2.2.4 thread 생성후, 생성한 쓰레드에서 malloc 호출 이전](#224-thread-생성후,-생성한-쓰레드에서-malloc-호출-이전)   
&nbsp;&nbsp;&nbsp;&nbsp;[2.2.5 생성한 쓰레드에서 malloc 호출 이후, free 호출 이전](#225-생성한-쓰레드에서-malloc-호출-이후,-free-호출-이전)   
&nbsp;&nbsp;&nbsp;&nbsp;[2.2.6 생성한 쓰레드에서 free 호출 이후](#226-생성한-쓰레드에서-free-호출-이후)   
[3. 참고자료](#3-참고자료)  


---

# 1. 데이터를 저장하는 여러 방법들

Heap을 설명하기 전, 일반적으로 데이터를 저장하는 방법들에 대해서 알아보자.

<br><br>

### 1.1 전역변수로 할당된 데이터

일반적으로 전역변수로 할당된 데이터는 컴파일 시점에서 사이즈가 결정된다.  아래 그림을 보면 전역변수로 64크기의 배열과 int형 변수 kkk가 선언되어있다.

<br>

![]({{ site.baseurl }}/images/heap/Heap%201/Untitled.png)

이러한 전역변수는 거의 모든 함수에서 사용하는 공통적인 데이터에 사용하며, lifetime 즉, 전역변수가 메모리에 상주하는 시간은 프로그램이 실행동안 남아있고, 프로그램이 종료되는 순간에 사라진다. 그리고 초기화 되지 않은 전역변수는 .bss 영역에 들어가고, 초기화 된 전역변수는 .data 영역에 들어간다.

- 초기화 된 kkk 변수

![]({{ site.baseurl }}/images/heap/Heap%201/Untitled%201.png)

<br>

- 초기화가 되지 않은 배열

![]({{ site.baseurl }}/images/heap/Heap%201/Untitled%202.png)

<br><br>

### 1.2 스택에 할당된 데이터

스택에 할당되는 데이터는 지역변수로 선언된 데이터들이 들어간다. 일반적으로 함수 안에서 선언한 데이터들이 지역변수에 해당된다. 아래 그램에서 a 변수가 지역변수에 해당된다. 해당 변수에 들어가는 정수 8은 스택영역에 저장된다.

![]({{ site.baseurl }}/images/heap/Heap%201/Untitled%203.png)

<br>

아래 그림을 보면 8데이터가 스택영역에 들어가있는 것을 확인 가능하다. 지역변수의 lifetime은 해당 지역변수가 선언된 함수가 return될때 해제된다. 

![]({{ site.baseurl }}/images/heap/Heap%201/Untitled%204.png)

<br><br>

### 1.3 동적으로 힙에 할당된 데이터

지역변수나 전역변수의 경우, 데이터 타입에 따라서 변수의 사이즈가 컴파일 시에 결정이된다. 이렇게 컴파일 시에 선언이 되는 변수의 사이즈는 고정값으로 변경하지 못한다.

하지만 컴파일 시가 아닌, 런타임 시에 변수의 사이즈가 결정되는 경우가 있는데 바로 힙영역이 이에 해당한다. 힙 영역에 데이터를 저장하게 되면, 이는 런타임 시에 데이터 사이즈가 결정되고, Lifetime 또한 런타임에만 알수 있다.( free 같은거)

![]({{ site.baseurl }}/images/heap/Heap%201/Untitled%205.png)

위 사진을 보면 사용자의 입력을 받는 만큼의 사이즈로 malloc을 진행하고 있다. 이부분이 바로 런타임시에 사용자의 입력에 따라서 사이즈가 달라지므로 컴파일시에는 사이즈를 알수 없다는 소리이다.

<br>

malloc이 반환하는 주소는 실제 데이터를 담을 수 있는 첫 시작 주소를 리턴하게되고, 이게 위 사진에서 test 포인터 변수에 들어가게 된다.

<br>

![]({{ site.baseurl }}/images/heap/Heap%201/Untitled%206.png)

위 사진이 실제 힙영역이다. 13라인의 5가 0x840247에 들어가 있는 것을 볼 수 있다. 바로 이 주소를 malloc이 수행되고 반환해주는 것이다.

추가적으로 예전 C언어를 배울때 가변 길이의 배열을 선언하지 못한다고 배웠었다. 

    		1. int b;
    
        2. scanf("%d",&b);
    
        3. char ar[b];
    
        4. scanf("%s",ar);
    
        5. printf("%s\n",ar);

예전에는 b 변수에 입력을 받고, 해당 사이즈만큼 ar 배열을 선언한다고 가정했을때, 컴파일을 하면 에러가 발생하였다.

하지만 C99 표준 이후로 가변길이 배열선언이 가능해졌고, 위와 같은 코드를 컴파일해도 정상적으로 동작한다고 한다. 자세한 내용은 아래 사이트에서 확인이 가능하다.

[C11 (C 버전)](https://ko.wikipedia.org/wiki/C11_(C_%EB%B2%84%EC%A0%84))

<br><br>

# 2. Heap 기초

<br>

## 2.1 Dynamic memory allocator

동적으로 할당된 메모리는 힙영역에서 관린된다고 위에서 설명했다. 이렇게 동적으로 할당된 메모리를 관리하기 위해서 dynamic memory allocator를 사용한다.

Allocator는 크게 두 종류로 나뉜다

- Explicit allocator : 개발자가 공간의 할당/해제를 관리  
    예) libc의 malloc과 free

- Implicit allocator : 개발자는 공간의 할당만 담당하고 free는 내부적으로 처리해줌  
    예) Java의 GC, Lisp 등

Explicit allocator에는 여러가지 종류가 있다. 간단하게 알아보자.

<br>

1. **dlmalloc**

    리눅스 초창기에 사용된 기본 메모리 할당자. dlmalloc에서 동일한 시간에 2개의 스레드가 malloc을 호출할 경우, freelist 데이터는 모두 사용가능한 스레드에 둘러싸인 상태로 공유되기 떄문에 오로지 하나의 쓰레드만 임계영역에 들어갈 수 있음. 이러한 이유로 다중  쓰레드 어플리케이션에서는 성능저하가 발생<br><br>

2. **ptmalloc2**

    dlmalloc에서 쓰레딩 지원 기능이 추가됨. ptmalloc2는 glibc 소스 코드에 통합됨. 동일한 시간에 2개의 쓰레드가 malloc을 호출할 경우, 메모리는 각각의 쓰레드가 분배된 힙 영역을 일정하게 유지하고, 힙을 유지하기 위한 freelist 데이터 또한 분배되어 있기 때문에 즉시 할당됨. <br><br>

3. **Jemalloc**

    Jason Evnas가 만들었고, 페이스북이나 파이어폭스에서 주로 사용함. 단편화 방지 및 동시 확장성을 강조한 할당자.<br><br>

4. **tcmalloc**

    구글이 만든 성능 도구에 포함되어 있는 힙 메모리 할당자로서 크롬 및 많은 프로젝트에서 사용. 멀티 쓰레드 환경에서 메모리 풀을 사용하는 속도를 개선하기 위한 목적으로 만들어졌다. 즉, 어플리케이션에서 따로 쓰레드 별로 메모리 풀 관리를 하지 않아도 되게 구현되있다. 

<br><br>

## 2.2 Glibc malloc (2.23기준)

gblic에서 사용하는 ptmalloc2에 대해서 자세히 알아보자. 특히 단순 익스를 위한 힙 공부가 아닌 힙의 구조를 이해하고 어떤식으로 현재 시스템에서 malloc이 실행되는 지를 위주로 공부를 하였다.

2.1 설명에서 ptmalloc2은 동일한 시간에 2개의 스레드가 malloc을 호출할 경우, 메모리는 각각의 스레드가 분배된 힙 영역을 일정하게 유지하고, 힙을 유지하기 위한 freelist data structures 또한 분배되어 있기 때문에 즉시 할당된다고 했다. 이렇게 각각의 스레드의 유지를 위해 분배된 힙과 freelist data structures의 행동을 **per thread arena**라고 부른다.

dlmalloc에서 쓰레드 기능이 추가된 것이 ptmalloc2이라고 하였는데, 쓰레드 관련 처리가 어떤식으로 진행되는지 코드로 알아보자.

<br>

- **코드**  

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>
#include <sys/types.h>

void* threadFunc(void* arg) {
    printf("Before malloc in thread 1\n");
    getchar();
    char* addr = (char*) malloc(1000);
    printf("After malloc and before free in thread 1\n");
    getchar();
    free(addr);
    printf("After free in thread 1\n");
    getchar();
}

int main() {
    pthread_t t1;
    void* s;
    int ret;
    char* addr;

    printf("Welcome to per thread arena example::%d\n",getpid());
    printf("Before malloc in main thread\n");
    getchar();
    addr = (char*) malloc(1000);
    printf("After malloc and before free in main thread\n");
    getchar();
    free(addr);
    printf("After free in main thread\n");
    getchar();
    ret = pthread_create(&t1, NULL, threadFunc, NULL);
    if(ret)
    {
            printf("Thread creation error\n");
            return -1;
    }
    ret = pthread_join(t1, &s);
    if(ret)
    {
            printf("Thread join error\n");
            return -1;
    }
    return 0;
}
```

<br>

코드에서 확인 할 부분은 다음과 같다.

1. main thread에서 malloc 호출 이전
2. main thread에서 malloc 호출 이후, free 호출 이전
3. main thread에서 free 호출 이후
4. thread 생성후, 생성한 쓰레드에서 malloc 호출 이전
5. 생성한 쓰레드에서 malloc 호출 이후, free 호출 이전
6. 생성한 쓰레드에서 free 호출 이후

6가지 상태에서 현재 힙 관리가 어떤식으로 진행되는지 확인해보자.

<br><br>

### 2.2.1) main thread에서 malloc 호출 이전

![]({{ site.baseurl }}/images/heap/Heap%201/Untitled%207.png)

현재 main() 에서 malloc이 호출되기 전 메모리 상태이다. pthread_create가 진행되지 않았으므로 메인 쓰레드 밖에 없는 것을 확인 할 수 있다. 또한 힙영역이 아직 없는 것을 확인할 수 있다

<br><br>

### 2.2.2) main thread에서 malloc 호출 이후, free 호출 이전

![]({{ site.baseurl }}/images/heap/Heap%201/Untitled%208.png)

malloc이 호출되고, free되기 전 상황이다. 힙 영역이 0x602000 ~ 0x623000 에 생성된 것을 확인할 수 있다. 이는 brk syscall을 사용하여 program break 위치를 증가시킴으로 확장된 영역이다.( 데이터 세그먼트 영역을 늘렸다는 뜻)

**"gef> catch syscall brk "** 명령어를 이용하여 brk에 bp를 걸고 malloc 함수가 호출되면 malloc이 실행되는 과정에서 brk syscall이 호출되는 것을 아래 그림에서 확인 할 수 있다

<br>

![]({{ site.baseurl }}/images/heap/Heap%201/Untitled%209.png)

또한, malloc 시 1000바이트만 요청했으나, 힙 메모리의 크기는 135168바이트(0x623000-0x602000)나 생성되었다. 이러한 힙 메모리의 인접한 지역을 **arena**라고 부른다. 이 **arena**는 메인 쓰레드로에 생성되었기 때문에 **main arena**라고 부른다.

이후, malloc을 통한 요청은 이 arena를 사용하여 해당 영역이 꽉찰때까지 지지고 볶으면서 사용한다. 만약 arena가 꽉찰 경우, 프로그램의 break 위치를 증가시킴으로써(데이터 세그먼트 영역을 확장시킨다는 뜻) 관리를 한다. 즉, Top chunk의 크기를 증가시켜 여분의 공간을 가질수 있도록 확장한다.

    **Top 청크** : arena의 최상단의 청크를 의미한다. 해당 설명은 아래에서 자세하기 하거나, 
    					다음 게시글에서 설명할 예정이다

<br><br>

### 2.2.3) main thread에서 free 호출 이후

![]({{ site.baseurl }}/images/heap/Heap%201/Untitled%2010.png)

위 사진을 보면 메모리 영역에 변화가 없는 것을 확인할 수 있다. free가 호출되어 메모리가 해제된 경우에, 즉각 운영체제에 반환되지 않았다는 소리이다. 할당된 메모리의 일부분(1000바이트) 오로지 **main_arena**의 bin에 이 해제된 청크를 추가하고, gblic malloc 라이브러리에 반환된다. 

이후, 사용자가 또다시 메모리를 요청하는 경우, 아까와 같이 brk syscall로 바로 할당해주는 것이 아닌, bin에 비어있는 블록이 있는지 탐색하고, 존재한다면 비어있는 블록을 할당해준다. 만약 비어있는 블록이 없다면 아까처럼 동일하게 메모리를 할당해준다.

    bin에 대한 설명 또한 다음 포스팅에서 자세하게 설명할 것이다.
    그냥 해제된 블록들을 관리하는 공간이라고만 알아두자.

<br><br>

### 2.2.4) thread 생성후, 생성한 쓰레드에서 malloc 호출 이전

![]({{ site.baseurl }}/images/heap/Heap%201/Untitled%2011.png)

pthread_create 함수를 통해 ID 값이 2인 쓰레드가 생성되었다. (thread2라 하자). thread2는 threadFunc 함수를 실행하는데 위 영역은 아직 malloc을 호출하지 않았으므로 thread2의 힙 영역은 없지만, 해당 **thread2의 쓰레드 스택**이 생성된 것을 확인 할 수 있다.

여기서 잠깐 쓰레드란 개념을 간단하고 보고가쟈

<br>

- **정의**
프로세스 내에서 실행되는 여러 흐름의 단위를 뜻함

- **특징**
    1. 스레드는 프로세스 내에서 각각 스택만 따로 할당받고, Code, Data, Heap 영역은 공유한다
    2. 스레드는 한 프로세스 내에서 동작되는 여러 실행의 흐름으로, 프로세스 내의 주소 공간이나 자원들(힙 공간 등)을 같은 프로세스 내에 스레드끼리 공유하면서 실행된다.
    3. 같은 프로세스 안에 있는 여러 스레드들은 같은 힙 공간을 공유한다. 반면에 프로세스는 다른 프로세스의 메모리에 직접 접근할 수 없다.
    4. 각각의 스레드는 별도의 레지스터와 스택을 갖고 있지만, 힙 메모리는 서로 읽고 쓸 수 있다.
    5. 한 스레드가 프로세스 자원을 변경하면, 다른 이웃 스레드(sibling thread)도 그 변경 결과를 즉시 볼 수 있다

<br>

![]({{ site.baseurl }}/images/heap/Heap%201/Untitled%2012.png)

따라서 위 사진처럼 code, data, heap 영역은 공유하지만, 각 쓰레드 별로 스택이 주어진다.  다시한번 thread2에서 malloc 호출 이전 상태를 확인해보자.

<br>

![]({{ site.baseurl }}/images/heap/Heap%201/Untitled%2011.png)

0x00007ffff6fef000 ~ 0x00007ffff77f0000 영역이 바로 thread2의 스택 영역이다

<br><br>

### 2.2.5) 생성한 쓰레드에서 malloc 호출 이후, free 호출 이전

![]({{ site.baseurl }}/images/heap/Heap%201/Untitled%2013.png)

![]({{ site.baseurl }}/images/heap/Heap%201/Untitled%2014.png)

thread2의 힙영역(0x00007ffff0000000 ~ 0x00007ffff4000000)이 생성된 것을 확인 할 수 있다.  해당 영역은 brk를 사용해서 할당하는 **main_thread**와는 달리 **mmap**을 사용하여 힙 메로리가 생성된다.  threadFunc 함수에서 malloc이 호출되는 과정중에 mmap이 호출된다. 아래 그림이 해당 mmap 호출 부분이다.

<br>

![]({{ site.baseurl }}/images/heap/Heap%201/Untitled%2015.png)

그리고 또한 여기서도 볼 수 있듯이, 사용자는 1000바이트만 요청했지만,  67MB 크기의 힙 메모리가 프로세스 주소 공간에 매핑되어있다. 이 67MB 중,135KB의 영역이 rw 권한으로 세팅되어, thread2를 위한 힙 메모리로 할당되었다. 이러한 메모리의 일부분(135KB)을 **thread_arena**라고 부른다

    참고로 사용자가 요청한 크기가 현재 arena(main_arena이든 thread_arene이든)에 사용자의 요청을 
    만족시킬 수 있는 충분한 공간이 없는 경우, **mmap syscall(brk 미사용)**을 사용하여 부족한 메모리를 할당한다.

<br><br>

### 2.2.6) 생성한 쓰레드에서 free 호출 이후

![]({{ site.baseurl }}/images/heap/Heap%201/Untitled%2016.png)

thread2에서 free 호출 이후 역시, 바로 메모리를 반환하지 않는다는 것을 확인 할 수 있다. 이 역시 아까처럼 해제한 영역은 thread_arena의 bin에 해제된 블럭을 추가하고 gblic malloc에 반환한다.

위와 같은 방법으로 메인 쓰레드, 쓰레드들이 관리가 된다. 추가적으로 

gblic malloc의 소스코드에서 다음과 같은 3개의 데이터 구조체를 확인 할 수 있다.

<br>

- **heap_info 구조체(Heap_Header)**  

```c
1.typedef struct _heap_info {
2.  mstate ar_ptr;    /* 현재 heap을 담당하고 있는 Arena */
3.  struct _heap_info *prev;  /* 이전 heap 영역 */
4.  size_t size;      /* 현재 size(bytes) */
5.  size_t mprotect_size; /* mprotected(PROT_READ|PROT_WRITE) size(bytes) */
6.  char pad[(-6*SIZE_SZ)&MALLOC_ALIGN_MASK]; /* 메모리 정렬 */
7.  /* sizeof(heap_info) + 2*SIZE_SZ는 MALLOC_ALIGN_MASK의 배수 */
} heap_info;
```


thread_arena는 각 쓰레드들에 대한 힙 영역이기 때문에 힙 영역의 공간이 부족하면 새로운 영역에 추가로 할당받기 때문에(mmap사용) 여러개의 힙 영역을 가질 수있다.

main_arena는 여러개의 힙을 가질수 없다. main_arena의 공간이 부족한 경우, sbrk 힙 영역은 메모리가 매핑된 영역까지 확장된다. 어쨋든 이런 힙 영역은 어떤 arena가 관리하고 있는지, 힙 영역의 크기가 어느정도인지, 이전에 사용하던 힙 영역의 정보가 어디에 있는지를 저장할 필요가 있다.

이런 정보를 저장하기 위한 구조체가 바로 위 구조체인 heap_info이며, 힙에 대한 정보를 저장하기 때문에 **Heap_Header**라고도 한다. (main_thread는 확장을 해서 쓰기 때문에 제외)

<br>

- **malloc_state 구조체(arena_Header)**    

        1.struct malloc_state
        2.{
        3.  /* Serialize access.  */
        4.  mutex_t mutex;
        5. 
        6.  /* Flags (formerly in max_fast).  */
        7.  int flags;
        8. 
        9.  /* Fastbins */
        10.  mfastbinptr fastbinsY[NFASTBINS];
        11. 
        12.  /* topchunk의 base address */
        13.  mchunkptr top;
        14. 
        15.  /* 가장 최근의 작은 요청으로부터 분리된 나머지 */
        16.  mchunkptr last_remainder;
        17. 
        18.  /* 위에서 설명한대로 pack된 일반적인 bins */
        19.  mchunkptr bins[NBINS * 2 - 2];
        20. 
        21.  /* Bitmap of bins */
        22.  unsigned int binmap[BINMAPSIZE];
        23. 
        24.  /* 연결 리스트 */
        25.  struct malloc_state *next;
        26. 
        27.  /* 해제된 아레나를 위한 연결 리스트 */
        28.  struct malloc_state *next_free;
        29. 
        30.  /* 현재 Arena의 시스템으로부터 메모리 할당  */
        31.  INTERNAL_SIZE_T system_mem;
        32.  INTERNAL_SIZE_T max_system_mem;
        33.};

<br>

위의 **Heap_Header**에서는 단순히 힙 영역에 대한 정보만을 저장하였다. arena는 힙 영역에 대한 정보 중에서도 어떤 부분을 사용하면 되는지를 관리하기 때문에 . 이를 알고 있을 필요가 있다. malloc_state 구조체는 각 arena에 하나씩 주어지고, 해제된 chunk를 관리하는 연결리스트 bin과 최상위 Chunk인 Top Chunk와 같은 arena에 대한 정보를 저장한다. 이를 **arena_header**라고도 부른다

단일 쓰레드 arena는 여러개의 힙을 가질 수 있지미나, 이러한 모든 힙에 대해서는 오직 하나의 arena_header만이 존재한다.

참고로 thread_arena와 달리, main_arena의 arena header는 sbrk 힙 영역의 일부가 아니다. main_arena는 **전역변수**이며, libc.so의 데이터 영역에서 찾을 수 있다.

<br>

- **malloc_chunk 구조체(Chunk Header)**

        1.struct malloc_chunk {
        2. 
        3.  INTERNAL_SIZE_T      prev_size;  /* Size of previous chunk (if free).  */
        4.  INTERNAL_SIZE_T      size;       /* Size in bytes, including overhead. */
        5. 
        6.  struct malloc_chunk* fd;         /* double links -- used only if free. */
        7.  struct malloc_chunk* bk;
        8. 
        9.  /* large block에서만 사용하고 해당 bin list의 크기 순서를 나타냄  */
        10.  struct malloc_chunk* fd_nextsize; /* double links -- used only if free. */
        11.  struct malloc_chunk* bk_nextsize;
        12.};

<br>

힙 영역은 사용자에 의해 할당되거나, 해제되거나 하면 Chunk라는 단위로 관리된다.

1) prev_size : 바로 이전 청크의 크기를 저장

2) size : 현재 청크크기를 저장

3) fd, bk : malloc시 데이터가 들어가고, free시 fd, bk포인터로 사용

4) fd(bk)_nextsize : large bin을 위해서 사용되는 포인터

    해당 청크의 구조에 대한 설명은 다음 포스팅에서 자세하게 다룰 예정

- **main_arena와 thread_arena를 그림으로 나타낸 경우(단일 힙 영역)**

![]({{ site.baseurl }}/images/heap/Heap%201/Untitled%2017.png)

<br>

- **thread_arena를 그림으로 나타낸 경우(여러 개의 힙 영역)**

    ![]({{ site.baseurl }}/images/heap/Heap%201/Untitled%2018.png)

<br><br>

## 2.3 함수 호출 과정 알고리즘

- **malloc 함수 호출 순서 : libc_malloc() → int_malloc() → sysmalloc()**
    1. libc_malloc() 함수에서 사용하는 Thread에 맞게 Arena를 설정한 후, int_malloc() 함수 호출
    2. int_malloc() 함수에서는 재사용할 수 있는 bin을 탐색하여 재할당하고, 마땅한 bin이 없다면 top chunk에서 분리해서 할당
    3. top chunk가 요청한 크기보다 작은 경우, sysmalloc() 함수 호출
    4. sysmalloc() 함수를 통해 시스템에 메모리를 요청해서 top chunk의 크기를 확장하고 대체

        *※ sysmalloc() 함수는 기존의 영역을 해제한 후, 새로 할당함*

    ![]({{ site.baseurl }}/images/heap/Heap%201/Untitled%2019.png)<br><br><br>

- **free 함수 호출 순서 :  libc_free() -> int_free() -> systrim() or heap_trim() or munmap_chunk()**
    1. libc_free() 함수에서 mmap으로 할당된 메모리인지 확인한 후, 맞을 경우 munmap_chunk() 함수를 통해 메모리 해제
    2. 아닌 경우, 해제하고자 하는 chunk가 속한 Arena의 포인터를 획득한 후, int_free() 함수 호출
    3. chunk를 해제한 후, 크기에 맞는 bin을 찾아 저장하고 top chunk와 병합을 할 수 있다면 병합 수행
    4. 병합된 top chunk가 너무 커서 Arena의 크기를 넘어선 경우, top chunk의 크기를 줄이기 위해 systrim() 함수 호출
    5. 문제가 없다면, heap_trim() 함수 호출
    6. mmap으로 할당된 chunk라면 munmap_chunk()를 호출<br><br>

    ![]({{ site.baseurl }}/images/heap/Heap%201/Untitled%2020.png)<br><br><br>

# 3. 참고자료

[https://tribal1012.tistory.com/78](https://tribal1012.tistory.com/78)

[https://tribal1012.tistory.com/141](https://tribal1012.tistory.com/141)

[http://egloos.zum.com/studyfoss/v/5206979](http://egloos.zum.com/studyfoss/v/5206979)