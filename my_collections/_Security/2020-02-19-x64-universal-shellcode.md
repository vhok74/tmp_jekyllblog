---
layout: post
title:  "window x64 환경  universal shellcode 구현"
date:   2020-02-19 19:45:55
image:  shellcode.png
tags:   [Window]
categories: [Computer]
---

**목차**   
[1. 소개](#1-소개)   
[2. 환경 구축](#2-환경-구축)   
[3. 분석](#3-분석)  
&nbsp;&nbsp;[3.1 Universal 쉘코드](#31-universal-쉘코드)  
&nbsp;&nbsp;[3.2 쉘코드 실행](#32-쉘코드-실행)   
[4. 결론](#4-결론)
- - -
# 1. 소개

" **윈도우 시스템 해킹 가이드 버그헌팅과 익스플로잇"**  개정판이 얼마전 나왔었다. 해당 챕터 중 Universal shellcode를 작성하는 부분이 존재하는데,  리눅스가 아닌 윈도우 환경에서는 쉘코드가 어떤식으로 동작하는 지를 공부하고자 시작하게되었다. 

추가적으로 책에서는 WinExe 함수를 이용하여 cmd창을 띄우는 과정을 설명하였지만, 이를 응용하여 x64 환경에서 MessageBoxA를 띄우는 것을 이번 목표로 세웠다.

---

# 2. 환경 구축

32bit 환경으로 책에 나온 예제를 1차적으로 공부하여 부족한 부분을 공부하고, 로컬 환경(x64)에서 실제 쉘코드 작성을 진행하였다. 32bit에서는 인라인 어셈블리로 작성가능했지만, 64bit에서는 불가능하다. 따라서 masm으로 따로 파일을 만들어서 cpp 파일에 링킹 시키는 과정으로 작성해야한다

1. 프로젝트 생성 - 콘솔 앱<br>  

2. 구성 관리자 변경 - 빌드 창 선택 - 구성관리자 선택
![]({{ site.baseurl }}/images/x64_shellcode/1.png)  
    
    
3. 프로젝트 우클릭 - 빌드 종속성 - 사용자 지정 빌드 - masm 체크
    ![]({{ site.baseurl }}/images/x64_shellcode/2.png)  
      
    ![]({{ site.baseurl }}/images/x64_shellcode/3.png)  
        
        
4. 이름.asm 파일 생성 - 해당 파일 우클릭 - 사진처럼 설정
    ![]({{ site.baseurl }}/images/x64_shellcode/4.png)  
         
         
5. asm 파일에서 함수명 지정
    ![]({{ site.baseurl }}/images/x64_shellcode/5.png)  
    
    
6. cpp 파일에서 해당 asm 링킹하여 사용
    ![]({{ site.baseurl }}/images/x64_shellcode/6.png)
    
    
- **32bit**  - 분석용
    1. 가상머신 : Virtual box
    2. 구동 OS : Window 7 home
    3. 디버거 : x32dbg 
    4. 개발 환경 : Visual studio 2010
    5. PEview

- **64bit - 실 테스트 용**
    1. 로컬 : 내 Host PC
    2. 구동 OS : Window 10  pro
    3. 디버거 : x64dbg
    4. 개발 환경 : Visual Studio 2019
    5. CFF Explorer

---
<br>
# 3. 분석

기존 책에서 설명하는 코드를 그대로 64bit 환경에서 적용하면 안된다. 레지스터의 활용 방법이나, 함수 호출규약, 인자 사용 방법 등 여러가지 변경된 부분을 알맞게 적용하여 쉘코드를 작성해야한다.

리눅스의 경우 system 함수를 바로 사용하여 원하는 기능을 수행하도록 쉘코드를 작성하면 되지만, 윈도우는 그렇게 간단하게 쉘코드를 작성할 수 없다.  여러가지 dll들이 링킹이 되고, 해당 dll안에 특정 함수를 실행시키 위해서는 이러한 과정을 다 직접 구현을 해줘야 하기 때문이다.

해당 분석 과정은 책에서 설명하는 **32bit** 기준과 **64bit**에서 다른 부분의 차이점을 위주로 설명을 할 것이다. 특히 쉘코드를 작성하면서 **64bit** 환경에서 시행착오를 한 부분을 위주로 작성을 할 것이다.    
<br><br>
  
## 3.1 Universal 쉘코드

프로세스에서 함수의 주소 값을 구하기 위해서 dll의 시작 주소 값과 dll 시작 주소부터 함수까지의 offset을 알아야 한다. 이 주소 값을 실행 중에 스스로 구해서 동적으로 입력해 주는 것이 바로 Universal 쉘코드의 목적이다

Universal 쉘코드의 원리를 이해하고 작성하기 위해서는 PEB, TEB, PE Header, Export Table, Export Name Table  등 몇 가지 추가적인 배경 지식이 필요하다  
  
<br>
### 1) TEB(Thread Environment Block)

TEB는 현재 실행되고 있는 쓰레드에 대한 정보를 담고 있는 구조체로, 각 쓰레드 별로 하나씩 TEB가 생성된다. 32bit에서는 TEB의 주소가 특수한 레지스터인 FS에 저장되어 있고, 64bit에서는 GS레지스터에 저장되어 있다

실제로 FS나 GS 레지스터에 TEB 주소값이 바로 들어있는 것은 아니며, TEB 주소를 가지고 있는 segment Descriptor Table의 Index 값을 가지고 있다고 한다

- **32bit**   
    ![]({{ site.baseurl }}/images/x64_shellcode/7.png)
*32bit 환경에서 TEB 구조체에 저장된 PEB 주소*   

- **64bit**   
    ![]({{ site.baseurl }}/images/x64_shellcode/8.png)
*64bit 환경에서 TEB 구조체에 저장된 PEB 주소*

windbg로 teb 구조체의 값을 직접 확인해 보았다. FS레지스터와 GS 레지스터를 이용하여 TEB에 접근할수 있고, TEB를 이용하여 PEB의 주소값을 알 수 있다.

32bit와 64bit의 차이점은 PEB의 주소값을 가지고 있는 ProcessEnviromentBlock 멤버변수가 위치하는 오프셋이 다르다는 점을 인지하고 있어야 한다.  

<br>
### 2) PEB(Process Enviroment Block)

PEB는 실행 중인 프로세스에 대한 정보를 담아두는 구조체이다. 프로세스와 관련된 다양한 정보들이 저장되어 있다. 이들 정보 중에는 프로세스에 로드 된 PE Image(EXE, DLL 등)에 대한 정보도 기록되어 있는데, 바로 이 정보를 이용하여 원하는 DLL 안에 존재하는 함수의 주소를 구할 것이다

- **32bit**   
    ![]({{ site.baseurl }}/images/x64_shellcode/9.png)
*PEB 구조체*   
  
- **64bit**   
    ![]({{ site.baseurl }}/images/x64_shellcode/10.png)
*PEB 구조체*

**Ldr 멤벼변수**의 위치 역시 32bit에서는 **0xc** 위치이고,  64bit에서는 **0x18** 위치로 차이가 있다.  요약해서 말하자면, Ldr이란 프로세스에 로드된 모듈에 대한 정보를 제공하는 PEB_LDR_DATA 구조체를 가리키는 포인터이다. **_PEB_LDR_DATA** 구조체를 살펴보자

  
<br>
### 3)PEB_LDR_DATA

PEB_LDR_DATA는 아래과 같은 구조로 구성되어 있는 구조체이다. 중간 부분에 **_LIST_ENTRY** 형 멤버변수를 3개 가지고 있는 것을 볼 수 있는데, 이 _LDR_ENTRY 는 더블 링크드 리스트 구조로 되어있는 구조체이다.

- **32bit**   
    ![]({{ site.baseurl }}/images/x64_shellcode/11.png)

- **64bit**   
    ![]({{ site.baseurl }}/images/x64_shellcode/12.png)

InLoadOrderModulList, InMemoryOrderModulList, InInitalizationOrderMoulduleLIst 는 각각 모듈의 로드된 순서, 메모리에 매핑된 순서, 초기화된 순서대로 각각 다른 더블링크드 리스트 형태로 구성되어 있다.

이해를 돕기위해 여기까지의 과정을 그림으로 표현하면 아래 사진과 같다

(사진은 32bit 기준으로 설명되었다)   
    ![]({{ site.baseurl }}/images/x64_shellcode/13.png)
*peb_ldr_data 구조도 / 출처 : [https://5kyc1ad.tistory.com/328](https://5kyc1ad.tistory.com/328)*

64bit도 오프셋을 제외하곤 차이가 없으므로 위 사진으로 설명을 하겠다.  맨처음 TEB를 이용하여 PEB 주소를 확인하였다. 그리고 PEB 구조체의 LDR 멤버변수를 확인하여 PEB_LDR_DATA에 접근할 수 있었다.

여기서 잠시 바로 **LDR_DATA_TABLE_ENTRY**에 대해서 설명하겠다. 이 구조체는 로드된 모듈에 대한 실제 정보 즉, PE정보들이 저장되어 있는 구조체이다. 예를 들어 ***notepade.exe***를 실행하면 메모장이 실행되기 위하여 실행되는 일련의 과정들이 있을 것이다.

**kernel32.dll, ntdll.dll, user32.dll** 등 등의 메모장을 구동하기 위해 필요한 **window api**들이 로딩이 되야 한다. LDR_DATA_TABLE_ENTRY는 이러한 모듈들의 PE 정보들을 저장하고 있는데,  각 모듈들끼리는 더블 링크드 리스트로 연결되어 있다. 

사진을 보면 각 모듈들의 정보가 담긴 ***LDR_DATA_TABLE_ENTRY*** 가 더블 링크드리스트로 서로서로를 가리키고 있는 것을 볼 수 있다. 이러한 ***LDR_DATA_TABLE_ENTRY***을 가리키고 있는 포인터가 바로 PEB_LDR_DATA 구조체에 저장되어있는 **InLoadOrderModulList, InMemoryOrderModulList, InInitalizationOrderMoulduleLIst** 이다.

따라서 이 더블 링크드리스트를 순회하여 원하는 dll을 확인 가능하고, 해당 구조체에서 DLLbase 주소 즉, DLL의 시작주소를 알 수 있는 것이다. 그렇다면 실제로 kernel32.dll 모듈의 base 주소를 확인 해보자.
    ![]({{ site.baseurl }}/images/x64_shellcode/14.png)
*peb_ldr_data 구조체의 InLoadOrderModuleList 의 Flink와 Blink*

해당 PEB_LDR_DATA 구조체에서 InLoadOrderModuleList를 봐보자. Flink와 Blink를 확인 할 수가 있는데, 여기서 Flink는 ldr_data_table_entry 구조체를 가리키고 있는 주소이다. Flink를 확인해보자.

![]({{ site.baseurl }}/images/x64_shellcode/15.png)

첫번째 LDR_DATA_TABLE_ENTRY 구조체는 실행 파일 그 자체에 대한 정보가 들어있다. (글 작성을 하다가 중간에 디버깅 대상을 바꿔서 notepad.exe가 아닌 test3.exe가 나와있다. 그냥 내가 실행시킨 프로그램 정보가 나온다고 알면 된다)

근데 여기서 궁금한 점이 생겼다. peb_ldr_data 구조체에서 Inorder.. 리스트의 Flink는 실행파일 그 자체에 대한 정보가 처음 리스트의 정보로 나온다고 했는데 그럼 Blink에는 무엇이 있을까라는 의문점이 들었다. 그래서 직접 한번 값을 찍어보았다.
    ![]({{ site.baseurl }}/images/x64_shellcode/16.png)
*peb_ldr_data의 InOrder...의 Blink 주소가 가리키는 ldr_data_table_entry*

해당 blink는 ***"ntmarta.dll"*** 모듈에 대한 LDR_DATA_TABLE_ENTRY 정보를 담고 있었다. 결론적이로 해당 모듈은 Flink를 따라가면서 마지막에 로드된 모듈이다. 해당 모듈이 나오고 다시 test3.exe가 나오는 것을 확인하였다.

따라서 peb_ldr_data의 inorder..의 Blink에는 마지막 모듈을 가리키고 있다고 판단하였다

![]({{ site.baseurl }}/images/x64_shellcode/17.png)   


요론식으로 말이다. 다시 본론으로 돌아가서 Flink를 계속 따라가다가 보면 Kernel32.dll 의 정보를 확인할 수 있었다.

![]({{ site.baseurl }}/images/x64_shellcode/18.png)
*Kernel32.dll의 Ldr_data_table_entry 구조체*

이렇게 우리가 원하는 Kernel32.dll 모듈의 DllBase 주소를 확인할 수 있다. 이 주소를 base로 하여 고정적 함수의 offset을 더하면 해당 dll 안에 속해있는 특정 함수의 주소를 구할 수 있다.

<br>

### 4) IMAGE_EXPORT_DIRECT

DLL은 자신이 어떤 함수들을 Export하고 있는지에 대한 정보를 PE 헤더에 저장하고 있으며, 이 정보는 PE헤더 ***IMAGE_OPTIONAL_HEADER64(32bit는 32)***의 DataDirectory 배열의 첫 번째 구조체인 Export Directory에 저장되어 있다

![]({{ site.baseurl }}/images/x64_shellcode/19.png)   


책에서 설명되어있는 image_optional_header는 32bit 기준으로 64bit는 약간 다르게 구성되어 있다. 아래의 사이트에서 해당 구조체의 멤버변수들을 확인 할 수 있다

[PE Format - Win32 apps](https://docs.microsoft.com/en-us/windows/win32/debug/pe-format#optional-header-standard-fields-image-only)

Image_optional_header의 DataDirectory 배열의 첫 번째 구조체인 Export Table은 IMAGE_EXPORT_DIRECTORY 구조체로 저장되어 있으며 아래와 같이 구성된다

![]({{ site.baseurl }}/images/x64_shellcode/66.png)  

이 역시 아래의 사이트에서 확인 가능하다

[PE Format - Win32 apps](https://docs.microsoft.com/en-us/windows/win32/debug/pe-format#section-table-section-headers)

EXPORT_DIRECOTRY에서 가장 중요한 멤버변수는 3가지이다<br>

- DWORD    AddressOfFunctions            :   실제 함수의 시작 주소까지의 오프셋 배열
- DWORD    AddressOfNames                :   함수 이름이 들어있는 배열
- DWORD    AddressOfNameORdinals    :   함수의 서수 배열(인덱스..?)

<br>

이 3가지 멤버변수를 사용하여 원하는 함수의 주소를 찾을 수 있다. 아래와 같은 과정을 거쳐 실제 함수 주소를 찾아보자

1. 함수명 배열로 이동해서 원하는 함수의 이름과 해당 인덱스를 찾는다
2. Ordinals 배열에서 인덱스에 해당하는 서수 인덱스 값을 찾는다
3. EAT 배열에서 서수 인덱스에 해당하는 함수 offset을 확인한다
4. DLLBase 주소와 offset을 더해서 함수의 실제 주소를 구한다

<br>

![]({{ site.baseurl }}/images/x64_shellcode/20.png)*단계별 Export 함수 주소 확인 과정*

<br>

간략하게 함수 주소를 구하는 과정을 설명하면 다음과 같다. 우선 Export Directory에서 AddressOfNames 부분을 뒤져서 원하는 함수를 찾는다. 여기서 원하는 함수명을 Hello라고 하자.

위 그림에서 Hello 함수는 3번째 위치 즉 인덱스 2에 위치해 있다. 그렇다면 AddressOfNameOrdinals의 2번째 인덱스에 들어 가있는 값을 확인해야한다. 2번째 인덱스에 들어있는 값은 2이다. 

이제 마지막으로 AddressOfFunctions(EAT)의 2번째 인덱스에 들어있는 값을 확인하자. 그림상 0x4aa5으로 확인이 되는데 이 값이 바로 base 주소 부터 Hello 함수가 떨어져 있는 오프셋을 뜻한다. 그렇다면 Hello 함수는 base+0x4aa5 의 주소로 접근하면 끝이다. 실제 디버거 상에서 확인해보자.
```
    0:004> dt nt!_image_optional_header64 00007fff`c3760000+0xe8+0x18
    ntdll!_IMAGE_OPTIONAL_HEADER64
       +0x000 Magic            : 0x20b
       +0x002 MajorLinkerVersion : 0xe ''
       +0x003 MinorLinkerVersion : 0xf ''
       +0x004 SizeOfCode       : 0x74800
       +0x008 SizeOfInitializedData : 0x38a00
       +0x00c SizeOfUninitializedData : 0
       +0x010 AddressOfEntryPoint : 0x17c70
       +0x014 BaseOfCode       : 0x1000
       +0x018 ImageBase        : 0x00007fff`c3760000
       +0x020 SectionAlignment : 0x1000
       +0x024 FileAlignment    : 0x200
       +0x028 MajorOperatingSystemVersion : 0xa
       +0x02a MinorOperatingSystemVersion : 0
       +0x02c MajorImageVersion : 0xa
       +0x02e MinorImageVersion : 0
       +0x030 MajorSubsystemVersion : 0xa
       +0x032 MinorSubsystemVersion : 0
       +0x034 Win32VersionValue : 0
       +0x038 SizeOfImage      : 0xb2000
       +0x03c SizeOfHeaders    : 0x400
       +0x040 CheckSum         : 0xbbbc4
       +0x044 Subsystem        : 3
       +0x046 DllCharacteristics : 0x4160
       +0x048 SizeOfStackReserve : 0x40000
       +0x050 SizeOfStackCommit : 0x1000
       +0x058 SizeOfHeapReserve : 0x100000
       +0x060 SizeOfHeapCommit : 0x1000
       +0x068 LoaderFlags      : 0
       +0x06c NumberOfRvaAndSizes : 0x10
       **+0x070 DataDirectory    : [16] _IMAGE_DATA_DIRECTORY**  
```   

<br>
base 주소에서 NT 헤더의 시작 부분 오프셋인 0xe8 + optional header의 오프셋 0x18을 더한 주소값을 이용하여 ***image_optional_header64*** 구조체의 값을 확인해 보았다.  이러한 값들은 32bit과 64bit에서 다르므로 잘 확인하여 더해줘야한다. 쨋든 맨 마지막 DataDirectory의 0번째 인덱스에 Export Directory 정보가 담겨져 있다.  
  

![]({{ site.baseurl }}/images/x64_shellcode/21.png)  
base주소에 저 0x8ec80 주소를 더하면 ***Image_export_directory***의 실제 주소를 알 수 있다
<br>
<br>
![]({{ site.baseurl }}/images/x64_shellcode/22.png)  

- dd base+0x8ec80 명령어 출력된 4바이트 데이터들과 **CFF Explorer**에서 Export **Directory** 값을 매칭시켜보면 동일하게 잘 출력된 것을 알 수 있다. **CFF** **Explorer**에서는 **Kernel32**.dll을 올려서 확인한 결과이다

 함수명 배열에서 **"ActivateActCtx"** 함수명이 위치해 있는 인덱스 값을 찾아보자. 우선 아까 찾은 ENT(Export Name Table)에 들어있는 값들을 확인해보자
![]({{ site.baseurl }}/images/x64_shellcode/23.png)  


0x9061c 오프셋에 **ENT** 주소가 들어있으므로 base에 더해서 출력한다음, 나오는 각 4바이트 값들을 da 명령어를 이용하여 함수명을 확인해보게되면, 3번째에 원하는 함수명이 들어있는 것을 알 수 있다

그럼이제 서수 테이블(EOT)에서 3번째 값(인덱스는 2)에 들어있는 값을 확인해보자
![]({{ site.baseurl }}/images/x64_shellcode/24.png)  


서수 테이블은 2바이트 단위로 보면된다. 여기서 2번째 인덱스의 값을 확인해보면 2가 들어가 있는 것을 알 수 있다. 이제 마지막으로 EAT 테이블의 2번째 인덱스를 확인해보면 아마 **ActiveActCtx** 함수의 실제 오프셋을 구할수 있을 것이다
![]({{ site.baseurl }}/images/x64_shellcode/25.png)  


EAT는 0x8eca8 오프셋에 위치해 있으므로 해당 부분을 dd 명령어를 이용하여 출력한뒤 인덱스2에 해당하는 값을 오프셋으로하여 u 명령어를 통해 출력을 해보면 원하는 함수명이 잘 출력되는 것을 알 수 있다. CFF Explorer에서 나오는 오프셋과 한번 비교를 해보자
![]({{ site.baseurl }}/images/x64_shellcode/26.png)  


ActiveActCtx 함수의 EAT인 Function RVA 영역을 보면 0x1E640 값이 들어가있는 것을 볼수있다. 이로서 정확하게 함수 주소의 오프셋을 구한것을 다시한번 확인하였다.

이제 동적으로 원하는 함수의 주소를 가져오는 건 끝났다. 해당 설명은 Kernel32.dll의 특정 함수를 가져오는 것에대해서 설명했지만, 다른 dll의 특정 함수도 같은 방법으로 구해오면 된다.

<br>

### 5) MessageBoxA 띄우기

책에서는 Kernel32.dll 모듈의 ExitProcess 함수, WinExec 함수의 주소를 구하여 cmd를 띄우는 과정을 설명하였다. 나는 여기서 응용하여 cmd가 아닌, Messagebox를 띄우는 것을 목표로 했기 때문에 추가적인 dll 모듈과 함수 주소를 구하는 과정을 추가해야한다. 다음의 단계로 수행을 할 것이다.

1. Kernel32.dll base 주소를 얻어온다
2. 1번을 이용하여 Exitprocess 함수, LoadLibrary 함수 들의 주소를 구한다
3. 2번에서 구한 LoadLibrary 함수를 이용하여 user32.dll의 base 주소를 구한다
4. 3번에서 구한 user32.dll base 주소를 이용하여 MessageBoxA 함수의 주소를 구한다
    (4번을 수행안하고 GetProcessAddress 함수를 이용하여 MessageBoxA 함수 주소를 구할 수 도 있다. 난 둘다 해봄)
5. MessageBoxA(0,'hello','hi',0) 함수를 실행한다
6. ExitProcess(0) 함수를 실행한다


<br>
- **kernel32.dll base 주소 구하기** 
```
    ; kernel32.dll base 주소 및 필요 주소값 구하는 과정
    xor rax, rax 
    xor rdi, rdi
    xor rsi, rsi
    xor rcx, rcx
    mov rax, gs : [rax+60h] ; peb
    mov rax, [rax + 18h]    ; peb_ldr_data
    mov rax, [rax + 10h]	; .exe inloadordermodulelist
    mov rbx, [rax]			; ntdll.dll inloadordermodulelist
    mov rbx, [rbx]			; kernel32.dll inloadordermodulelist
    mov rbx, [rbx + 30h] 	; kernel32.dll base adr 
    
    mov edi, dword ptr [rbx + 3ch] ; pe header
    add rdi, rbx 
    xor r8,r8
    add r8,rdi
    add r8,40h
    mov edi, dword ptr [r8 + 48h]  ; Export Table
    add rdi, rbx 
    mov[rbp + 18h], rdi  
    mov esi, dword ptr [rdi + 20h] ; Export Name Table
    add rsi, rbx 
    mov ecx, dword ptr [rdi + 24h] ; Ordinal Table
    add rcx, rbx         
    xor rdx, rdx 
    
    push rax 
    push rcx 
    push rdx 
    push rbx 
    push rsp 
    push rbp 
    push rsi 
    push rdi
```


코드에 대한 설명은 위에서 다 했으니 생략하겠다. 해당 과정을 통하여 kernel32.dll의 base 주소와 EAT, ENT, Ordinal Table 의 주소를 구할 수 있고, 이를 이용하여 Exitprocess, LoadlibraryA 함수 주소를 구할 것이다. 참고로 마지막에 레지스터들을 push하는 이유는 현재 상태의 레지스터 값을 저장하기위해서 수행한 것이다. 책에서 ***pushad*** 명령어의 역할을 한다고 보면된다. (64bit에는 ***pushad*** 없음)

<br>

- Exitprocess 함수, LoadLibrary 함수, user32.dll의 base 주소구하기
```
    xor rdi, rdi 
    add di, 479h 
    call **get_func_addr**   ; exitprocess 주소 얻기
    mov[rbp + 40h], rax  ; exitprocess 주소 저장
    
    pop rdi
    pop rsi
    pop rbp 
    pop rsp 
    pop rbx 
    pop rdx 
    pop rcx 
    pop rax 
    
    xor rdi, rdi 
    add di, 496h	
    call **get_func_addr**   ; loadlibrary 주소 얻기
    mov[rbp + 48h], rax  ; loadlibrary 주소 저장
    
    sub rsp, 8h
    xor rax, rax
    mov qword ptr[rbp+20h],rax
    mov qword ptr[rbp+28h],rax
    mov byte ptr[rbp+20h],75h  ; u
    mov byte ptr[rbp+21h],73h  ; s
    mov byte ptr[rbp+22h],65h  ; e
    mov byte ptr[rbp+23h],72h  ; r
    mov byte ptr[rbp+24h],33h  ; 3
    mov byte ptr[rbp+25h],32h  ; 2
    mov byte ptr[rbp+26h],2eh  ; .
    mov byte ptr[rbp+27h],64h  ; d
    mov byte ptr[rbp+28h],6ch  ; l
    mov byte ptr[rbp+29h],6ch  ; l
    lea rcx, [rbp+20h]
    call qword ptr[rbp + 48h]  ; call loadlibrary
```
<br>

함수의 주소를 구하는 과정은 책에서 설명하는 과정과 동일하게 진행하였다. 간단하게 요약하자면 파이썬 코드로 원하는 dll에 존재하는 각 함수명의 해시값을 계산하여 출력한뒤, 원하는 함수의 해시값을 가져온다.

그다음 get_func_addr 레이블로 이동하여 dll에 존재하는 함수명의 해시값을 계산하고 이와 저장한 해시값을 비교하여 동일한 값이 나올때까지 반복한다.
```
EX) hash(WinExec) = W + i + n + E + x + e + c
                              = 0x57 + 0x69 + 0x6e + 0x45 + 0x78 + 0x65 + 0x63
                              = 0x2b3
```
필요한 해시값은 다음과 같다

- ***Exitprocess*** : 0x479
- ***LoadlibraryA*** : 0x496
- ***MessageBoxA*** : 0x42f

<br>

그렇다면 정상적으로 LoadLibraryA 함수가 나오는지 확인해보자.
![]({{ site.baseurl }}/images/x64_shellcode/27.png)  


현재 rax에는 LoadlibrayA 함수의 주소가 저장되어있다. kernel32.dll의 base 주소는 0x7FFA63660000이기때문에 rax에서 base를 빼면 ***00007FFA6367EB60h- 00007FFA63660000h = 1EB60h*** 이다. CFF Explorer에서 확인해보자

<br>
![]({{ site.baseurl }}/images/x64_shellcode/28.png)  


오프셋이 정확히 맞아 떨어졌다. 이를 통해 제대로 값을 가져온 것을 알수있다.

이제 user32.dll의 base 주소를 구했으니 동일한 방법으로 ***MessageBoxA*** 함수 주소를 구하고 적당한 인자를 줘서 호출하면 끝이다

<br>

    ; user32.dll의 EAT, ENT, Ordinal Table 주소 구하기
    		xor rdi, rdi
    		xor rsi, rsi
    		xor rcx, rcx
    		mov rbx, rax 	; rax는 user32.dll base adr 
    		mov edi, dword ptr [rbx + 3ch]   
    		add rdi, rbx 
    		xor r8,r8
    		add r8,rdi
    		add r8,40h
    		mov edi, dword ptr [r8 + 48h]  
    		add rdi, rbx 
    		mov[rbp + 18h], rdi  
    		mov esi, dword ptr [rdi + 20h]  
    		add rsi, rbx 
    		mov ecx, dword ptr [rdi + 24h] 
    		add rcx, rbx         
    		xor rdx, rdx 
    		xor rdi, rdi
    		add di, 42fh
    		call get_func_addr  ; MessageBoxA 함수 주소 구하기
    		mov[rbp + 50h], rax ;

주의해야 할 것은 함수를 호출할때 윈도우 함수 호출 규약에 따라서 호출해야한다. 즉 첫번째 인자부터 4번째 인자까지는  rcx, rdx, r8, r9 로 전달되고, 그 이상의 인자는 스택에 올려져서 호출해야하는게 윈도우 64bit의 규약이다. 해당 설명이 잘 나와있는 곳이 있어서 참고하면 좋을듯 싶다
[[시스템] 윈도우 x64 호출 규약 리뷰](http://www.jiniya.net/ng/2017/11/x64-calling-convention/)
<br><br>
## 3.2 쉘코드 실행

이렇게 최종적으로 작성된 쉘코드는 다음과 같다.

    .code
    
    SHELL PROC
    jmp start
    
    get_func_addr :
    loop_ent:
    		inc rdx  
    		mov eax,dword ptr [rsi] 
    		add rsi,4 
    		push rax 
    		push rcx 
    		push rdx 
    		push rbx 
    		push rsp 
    		push rbp 
    		push rsi 
    		push rdi 
    		add	rbx, rax 
    		mov	rsi, rbx 
    		xor rax, rax 
    		xor rdi, rdi 
     hash :
    		mov al, byte ptr[rsi] 
    		add rsi, 1 
    		add rdi, rax   
    		test al, al 
    		jnz hash 
    		mov qword ptr [rbp + 10h], rdi  
    		pop rdi
    		pop rsi
    		pop rbp 
    		pop rsp 
    		pop rbx 
    		pop rdx 
    		pop rcx 
    		pop rax 
    		cmp[rbp + 10h], rdi  
    		jne loop_ent 
    		movzx rdx, word ptr[rcx + rdx * 2 - 2] 
    		mov rdi, [rbp + 18h]
    		xor rsi,rsi
    		mov esi, dword ptr [rdi + 1ch]   
    		mov rdi, rbx 
    		add rsi, rdi 
    		xor rbx,rbx
    		mov ebx, dword ptr [rsi + rdx * 4] 
    		add rdi,rbx
    		mov rax, rdi 
    		ret 
    
    start:
    		xor rax, rax 
    		xor rdi, rdi
    		xor rsi, rsi
    		xor rcx, rcx
    		mov rax, gs : [rax+60h]
    		mov rax, [rax + 18h]    
    		mov rax, [rax + 10h]  
    		mov rbx, [rax]       
    		mov rbx, [rbx]       
    		mov rbx, [rbx + 30h] 	**; kernel32.dll base adr** 
    		mov edi, dword ptr [rbx + 3ch]   
    		add rdi, rbx 
    		xor r8,r8
    		add r8,rdi
    		add r8,40h
    		mov edi, dword ptr [r8 + 48h]  
    		add rdi, rbx 
    		mov[rbp + 18h], rdi  
    		mov esi, dword ptr [rdi + 20h]  
    		add rsi, rbx 
    		mov ecx, dword ptr [rdi + 24h] 
    		add rcx, rbx         
    		xor rdx, rdx 
    		push rax 
    		push rcx 
    		push rdx 
    		push rbx 
    		push rsp 
    		push rbp 
    		push rsi 
    		push rdi 
    		xor rdi, rdi 
    		add di, 479h 
    		call get_func_addr   **; exitprocess 주소 얻기**
    		mov[rbp + 40h], rax  **; exitprocess 주소 저장**
    		pop rdi
    		pop rsi
    		pop rbp 
    		pop rsp 
    		pop rbx 
    		pop rdx 
    		pop rcx 
    		pop rax     
    		xor rdi, rdi 
    		add di, 496h	
    		call get_func_addr   **; loadlibrary 주소 얻기**
    		mov[rbp + 48h], rax  **; loadlibrary 주소 저장**   		
    		sub rsp, 8h
    		xor rax, rax
    		mov qword ptr[rbp+20h],rax
    		mov qword ptr[rbp+28h],rax
    		mov byte ptr[rbp+20h],75h  ; u
    		mov byte ptr[rbp+21h],73h  ; s
    		mov byte ptr[rbp+22h],65h  ; e
    		mov byte ptr[rbp+23h],72h  ; r
    		mov byte ptr[rbp+24h],33h  ; 3
    		mov byte ptr[rbp+25h],32h  ; 2
    		mov byte ptr[rbp+26h],2eh  ; .
    		mov byte ptr[rbp+27h],64h  ; d
    		mov byte ptr[rbp+28h],6ch  ; l
    		mov byte ptr[rbp+29h],6ch  ; l
    		lea rcx, [rbp+20h]
    		call qword ptr[rbp + 48h]  ; call loadlibrary
 
    		; user32.dll의 EAT, ENT, Ordinal Table 주소 구하기
    		xor rdi, rdi
    		xor rsi, rsi
    		xor rcx, rcx
    		mov rbx, rax 	**; rax는 user32.dll base adr** 
    		mov edi, dword ptr [rbx + 3ch]   
    		add rdi, rbx 
    		xor r8,r8
    		add r8,rdi
    		add r8,40h
    		mov edi, dword ptr [r8 + 48h]  
    		add rdi, rbx 
    		mov[rbp + 18h], rdi  
    		mov esi, dword ptr [rdi + 20h]  
    		add rsi, rbx 
    		mov ecx, dword ptr [rdi + 24h] 
    		add rcx, rbx         
    		xor rdx, rdx 
    		xor rdi, rdi
    		add di, 42fh
    		call get_func_addr  **; MessageBoxA 함수 주소 구하기**
    		mov[rbp + 50h], rax     
    		sub rsp, 20h 
    		xor rax, rax 
    		xor rcx, rcx 
    		xor rdx, rdx 
    		mov qword ptr[rbp+28h],rax 
    		mov byte ptr[rbp+28h],68h 
    		mov byte ptr[rbp+29h],65h 
    		mov byte ptr[rbp+2ah],6ch 
    		mov byte ptr[rbp+2bh],6ch  
    		mov byte ptr[rbp+2ch],6fh  
    		xor r8, r8
    		mov qword ptr[rbp+30h],rcx
    		mov byte ptr[rbp+30h], 68h 
    		mov byte ptr[rbp+31h], 69h  
    		xor r9, r9
    		lea rdx, [rbp+28h]
    		lea r8, [rbp+30h] 
    		call qword ptr[rbp+50h]   **; call messageBoxA(0,'hello','hi',0)**   
    		xor rcx, rcx  
    		call qword ptr[rbp + 40h] **; call exitprocess**  
    SHELL ENDP	
    End
<br>

실행시키면 MessageBoxA가 실행되는 것을 확인 할 수 있다
![]({{ site.baseurl }}/images/x64_shellcode/29.png)  


이제 바이트 코드를 추출하여 쉘코드를 완성해보자. 해당 쉘코드는 너무 길어 visual studio의 디스어셈블러에서 바이트코드를 추출하기는 애매하다. 따라서 아이다의 ***LazyIda*** 플러그인을 이용하여 배열 형태로 바이트코드를 추출하였다. 아이다에서 쉘코드 영역을 드래그한 후

<br>
![]({{ site.baseurl }}/images/x64_shellcode/30.png)  


빨간색으로 쳐진 부분을 누르면 아래 부분에 쉘코드가 배열 형태로 변환된다.

<br>
![]({{ site.baseurl }}/images/x64_shellcode/31.png)  


 저 배열을 복사하여 쉘코드로 만든뒤 다음과 같이 코드를 작성하였다.   
![]({{ site.baseurl }}/images/x64_shellcode/32.png)  

그리고 실행을 하면

![]({{ site.baseurl }}/images/x64_shellcode/33.png)  


정상적으로 쉘코드가 실행된 것을 확인 할 수 있다. 쉘코드를 자세히 보면 널바이트가 하나도 없다. 왜냐하면 직접 널바이트 제거를 위한 작업을 수행했기 때문이다. 그 이후 책을 보니 쉘코드 인코딩을 이용하여 널바이트 제거와 bad char 제거에 대한 내용이 설명되어있다.

이 부분을 알지 못하고 직접 구글링하여 널바이트를 제거했는데 얼추 잘 제거된거같다. 다음 포스팅에는 인코딩 부분과 관련하여 작성할 예정이다.


# 4. 결론

윈도우와 관련하여 공부는 이번이 처음이였다. 그리고 어셈블리어를 이용하여 코딩도 해보고 이런적이 없었는데 이번을 통해 많은 공부가 되었다. 앞으로 공부할 때도 하나를 알더라도 완벽히 이해하고 넘어가는게 좋은 것같다. 끝  
