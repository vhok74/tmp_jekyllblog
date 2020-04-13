---
layout: post
title:  "stdout의 file structure flag를 이용한 libc leak"
date:   2020-04-04 19:45:55
image:  stdout.PNG
tags:   [Pwnable]
categories: [Security]
---

hackctf childheap 문제를 풀면서 알게된 libc 주소 leak하는 방법에 대해서 설명하겠다. 이 방법은 libc leak을 하기위한 여러 공격 벡터중 하나이다.

<br>

## 1. leak 시나리오

해당 방법은 ```_IO_FILE struct```의 flag값을 임의로 변경하고, _```IO_write_base``` 의 일부를 NULL로 변조함으로써 stdout을 사용하는 함수가 호출될때 비정상적인 루틴을 통해 libc가 leak이 되도록 한다

<br>

---

## 2. _IO_FILE_struct

내부적으로 stdout이 언제 어떻게 사용되는지 간단한 예시를 통해 이해해보자.

1. puts("hello")가 호출됨
2. puts함수 루틴이 실행됨
    - 결국 내부적으로 ```write_sys``` 를 호출하여 출력이 되는데 어떠한 문자열을 출력할 때마다 커널버퍼를 사용하면 매우 비효율적임
    - 따라서 glibc 내부의 임시버퍼에 해당 문자열을 임시로 저장하는 방식을 이용함
    - 이때 ```_IO_2_1_stdout``` 를 이용하여 임시출력버퍼에 저장하는 로직을 거침
    - (BUT ! 해당 바이너리를 setvbuf 함수로 임시버퍼를 사용하지 않게 설정되어있음. 따라서 실제 버퍼의 주소가 대신 들어감. 참고하길)
    - ```_IO_2_1_stdout``` 는 ```_IO_FILE``` 구조체 타입으로써 아래의 형태를 띈다<br><br>

            struct _IO_FILE {
              **int _flags;**           /* High-order word is _IO_MAGIC; rest is flags. */
            #define _IO_file_flags _flags
            
              /* The following pointers correspond to the C++ streambuf protocol. */
              /* Note:  Tk uses the _IO_read_ptr and _IO_read_end fields directly. */
              char* _IO_read_ptr;   /* Current read pointer */
              char* _IO_read_end;   /* End of get area. */
              char* _IO_read_base;  /* Start of putback+get area. */
              char* _IO_write_base; /* Start of put area. */
              char* _IO_write_ptr;  /* Current put pointer. */
              char* _IO_write_end;  /* End of put area. */
              char* _IO_buf_base;   /* Start of reserve area. */
              char* _IO_buf_end;    /* End of reserve area. */
              /* The following fields are used to support backing up and undo. */
              char *_IO_save_base; /* Pointer to start of non-current get area. */
              char *_IO_backup_base;  /* Pointer to first valid character of backup area */
              char *_IO_save_end; /* Pointer to end of non-current get area. */
            
              struct _IO_marker *_markers;
              ...

        결국 ```_IO_2_1_stdout``` 는 위와 같은 구조체의 형태로 puts함수 내부에서 사용되는데 우리가 주목해야할 멤버변수는 ```_flags``` 이다. 이는 읽기 전용, 쓰기 전용 등의 권한을 설정하는 flag 이다. 기본적으론 **0xfbad0000** 가 기본 매직값이고, 하위 2비트는 여러 플래그들이 들어갈 수 있다.<br><br>

    - _flags 값

            ...
            /* Magic numbers and bits for the _flags field.
               The magic numbers use the high-order bits of _flags;
               the remaining bits are available for variable flags.
               Note: The magic numbers must all be negative if stdio
               emulation is desired. */
            
            **#define _IO_MAGIC 0xFBAD0000 /* Magic number */**
            #define _OLD_STDIO_MAGIC 0xFABC0000 /* Emulate old stdio. */
            #define _IO_MAGIC_MASK 0xFFFF0000
            #define _IO_USER_BUF 1 /* User owns buffer; don't delete it on close. */
            #define _IO_UNBUFFERED 2
            #define _IO_NO_READS 4 /* Reading not allowed */
            #define _IO_NO_WRITES 8 /* Writing not allowd */
            #define _IO_EOF_SEEN 0x10
            #define _IO_ERR_SEEN 0x20
            #define _IO_DELETE_DONT_CLOSE 0x40 /* Don't call close(_fileno) on cleanup. */
            #define _IO_LINKED 0x80 /* Set if linked (using _chain) to streambuf::_list_all.*/
            #define _IO_IN_BACKUP 0x100
            #define _IO_LINE_BUF 0x200
            #define _IO_TIED_PUT_GET 0x400 /* Set if put and get pointer logicly tied. */
            #define _IO_CURRENTLY_PUTTING 0x800
            #define _IO_IS_APPENDING 0x1000
            #define _IO_IS_FILEBUF 0x2000
            #define _IO_BAD_SEEN 0x4000
            #define _IO_USER_LOCK 0x8000
            
            #define _IO_FLAGS2_MMAP 1
            #define _IO_FLAGS2_NOTCANCEL 2
            #ifdef _LIBC
            ...

        <br>flags는 4바이트 크기이다. 여기서 상위 2바이트는 ```magic``` 이라는 이름으로 0xFBAD???? 값을 가지고 있다. 이 부분은 고정값이다. 이제 하위 2바이트를 위 상수로 정의된 것들을 조합하여 구성하게 된다.<br><br>

        이를 이용하여 FILE 구조체와 관련된 설정을 추가할 수 있다. 만약 read_only로 설정된 권한의 FILE stream이 설정되면 위 상수값에서  ```NO_READS``` 값이 추가될 것이다.
    
        어쨋든 일반적으로 puts가 호출되어 stdout 의 flag 값은 다음과 같이 세팅되어 있다<br><br>
    
        ![]({{ site.baseurl }}/images/study/pwnable/stdout%20file%20structure%20flag%20libc%20leak/Untitled.png)
    
        1. _IO_MAGIC = ```0xfbad```
        2. _IO_IS_FILEBUF = ```0x2000```
        3. _IO_CURRENTLY_PUTTING = ```0x800```
        4. _IO_LINKED = `0x80` 
        5. _IO_NO_READS &#124; _IO_UNBUFFERED &#124; _IO_USER_BUF    
            ⇒  4   &#124;   2    &#124;    1  = ```7```
    
        6. Total flag = ```0xfbad0000```  + ```0x2000``` + ```0x800``` + ```0x80```  + ```7```
                               &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;= `0xfbad2887` 
    
        5번을 보면 UNBUFFERED와 USER_BUF가 설정되는 것으로 보아 아까 설명했던 setvbuf함수로 임시버퍼를 사용하지 않는다는 것을 알 수 있다.
    
        쨋든 이러한 플래그 값들은 puts함수 내부 루틴 중 _IO*_*new_do_write 함수가 호출될때 첫번째 인자로 들어가게된다. 이는 파일 포인터 자리로 매직값+플래그가 설정된 stdout 값이다.
    
        여기서 우리는 stdout 구조체 변수 값인 `0xfbad2887` 에서 `_IO_APPENDING` 가 포함되게 값을 변경시켜야 한다. (이유는 아직 알지 못했다..)
    
        해당 flag 값을 변경했다면, 아래의 멤버변수를 이용하면 leak을 할 수 있다.
    
        ![]({{ site.baseurl }}/images/study/pwnable/stdout%20file%20structure%20flag%20libc%20leak/Untitled%201.png)
    
        어떻게 저 멤버변수를 이용해서 leak을 할 수 있는지는 puts함수의 내부 루틴을 확인하면 이해가 된다.

<br>

---
<br><br>
## 3. puts 함수 내부 루틴

1. **_IO_PUTS 함수**

        #include "libioP.h"
        #include <string.h>
        #include <limits.h>
        int
        _IO_puts (const char *str)
        {
          int result = EOF;
          size_t len = strlen (str);
          _IO_acquire_lock (stdout);
          if ((_IO_vtable_offset (stdout) != 0
               || _IO_fwide (stdout, -1) == -1)
              && **_IO_sputn (stdout, str, len)** == len
              && _IO_putc_unlocked ('\n', stdout) != EOF)
            result = MIN (INT_MAX, len + 1);
          _IO_release_lock (stdout);
          return result;
        }
        weak_alias (_IO_puts, puts)
        libc_hidden_def (_IO_puts)

    - _IO_PUTS 함수 내부에서 ```_IO_sputn``` 이 호출된다. 이는 원하는 길이만큼 출력하기 위해서 호출되는 함수이다
    - 해당 함수는 매크로 형태로 정의되어 있다.<br><br><br>

2. **_IO_sputn 함수**

        #define _IO_sputn(__fp, __s, __n) _IO_XSPUTN (__fp, __s, __n) 
        #define _IO_XSPUTN(FP, DATA, N) JUMP2 (__xsputn, FP, DATA, N) 
        #define JUMP2(FUNC, THIS, X1, X2) (_IO_JUMPS_FUNC(THIS)->FUNC) (THIS, X1, X2) 
        #define _IO_JUMPS_FUNC(THIS) \ 
        	(IO_validate_vtable \ 
        	(*(struct _IO_jump_t **) ((void *) &_IO_JUMPS_FILE_plus (THIS) \ 
        							+ (THIS)->_vtable_offset))) 
        
        #define _IO_JUMPS_FILE_plus(THIS) \ 
        	_IO_CAST_FIELD_ACCESS ((THIS), struct _IO_FILE_plus, vtable) 
        
        #define _IO_CAST_FIELD_ACCESS(THIS, TYPE, MEMBER) \ 
        	(*(_IO_MEMBER_TYPE (TYPE, MEMBER) *)(((char *) (THIS)) \ 
        																		+ offsetof(TYPE, MEMBER))) 
        #define _IO_MEMBER_TYPE(TYPE, MEMBER) __typeof__ (((TYPE){}).MEMBER) 
        
        출처: https://nekoplu5.tistory.com/229 [NekoPlus_]

    - _IO_sputn 함수는 위와같이 매크로 형태로 정의되어 있다. 아래의 매크로로 쭉 이어지는데
    - vtable에 어떠한 멤버변수들이 있는지 확인해보자<br><br><br>

3. **stdout->vtable**

    ![]({{ site.baseurl }}/images/study/pwnable/stdout%20file%20structure%20flag%20libc%20leak/Untitled%202.png)

    - 해당 멤버변수들 중 호출되는 것은 __xsputn이다. 해당 값에는 ```_IO_new_file_xsputn``` 함수가 설정되어있으므로 해당 함수가 호출될 것이다. 이 함수를 봐보자<br><br><br>

4. **_IO_new_file_xsputn 함수**

        _IO_new_file_xsputn (FILE *f, const void *data, size_t n)
        {
          const char *s = (const char *) data;
          size_t to_do = n;
          int must_flush = 0;
          size_t count = 0;
          if (n <= 0)
            return 0;
          /* This is an optimized implementation.
             If the amount to be written straddles a block boundary
             (or the filebuf is unbuffered), use sys_write directly. */
          /* First figure out how much space is available in the buffer. */
          if ((f->_flags & _IO_LINE_BUF) && (f->_flags & _IO_CURRENTLY_PUTTING))
            {
              count = f->_IO_buf_end - f->_IO_write_ptr;
              if (count >= n)
                {
                  const char *p;
                  for (p = s + n; p > s; )
                    {
                      if (*--p == '\n')
                        {
                          count = p - s + 1;
                          must_flush = 1;
                          break;
                        }
                    }
                }
            }
          else if (f->_IO_write_end > f->_IO_write_ptr)
            count = f->_IO_write_end - f->_IO_write_ptr; /* Space available. */
          /* Then fill the buffer. */
          if (count > 0)
            {
              if (count > to_do)
                count = to_do;
              f->_IO_write_ptr = __mempcpy (f->_IO_write_ptr, s, count);
              s += count;
              to_do -= count;
            }
          if (to_do + must_flush > 0)
            {
              size_t block_size, do_write;
              /* Next flush the (full) buffer. */
              if (**_IO_OVERFLOW (f, EOF)** == EOF)
                /* If nothing else has to be written we must not signal the
                   caller that everything has been written.  */
                return to_do == 0 ? EOF : n - to_do;
        ...

    - 출력할 문자의 길이 + must_flush의 합이 0보다 크면 이전에 출력한 크기 이상으로 문자열 출력이 왔다는 뜻이므로 ```_IO_overflow```  함수를 호출하게 된다.
    - 해당 함수도 매크로로 설정되어 있다.

            #define _IO_OVERFLOW(FP, CH) JUMP1 (__overflow, FP, CH)

    - 아까 확인했던 구조체 멤버변수중 해당 함수는 __overflow를 가지기 때문에 ```_IO_new_file_overflow``` 함수가 호출될 것이다.<br><br><br>

5. **_IO_new_file_overflow  함수**

        _IO_new_file_overflow (FILE *f, int ch)
        {
          if (f->_flags & _IO_NO_WRITES) /* SET ERROR */
            {
              f->_flags |= _IO_ERR_SEEN;
              __set_errno (EBADF);
              return EOF;
            }
          /* If currently reading or no buffer allocated. */
          if ((f->_flags & _IO_CURRENTLY_PUTTING) == 0 || f->_IO_write_base == NULL)
            {
              /* Allocate a buffer if needed. */
              if (f->_IO_write_base == NULL)
                {
                  _IO_doallocbuf (f);
                  _IO_setg (f, f->_IO_buf_base, f->_IO_buf_base, f->_IO_buf_base);
                }
              /* Otherwise must be currently reading.
                 If _IO_read_ptr (and hence also _IO_read_end) is at the buffer end,
                 logically slide the buffer forwards one block (by setting the
                 read pointers to all point at the beginning of the block).  This
                 makes room for subsequent output.
                 Otherwise, set the read pointers to _IO_read_end (leaving that
                 alone, so it can continue to correspond to the external position). */
              if (__glibc_unlikely (_IO_in_backup (f)))
                {
                  size_t nbackup = f->_IO_read_end - f->_IO_read_ptr;
                  _IO_free_backup_area (f);
                  f->_IO_read_base -= MIN (nbackup,
                                           f->_IO_read_base - f->_IO_buf_base);
                  f->_IO_read_ptr = f->_IO_read_base;
                }
              if (f->_IO_read_ptr == f->_IO_buf_end)
                f->_IO_read_end = f->_IO_read_ptr = f->_IO_buf_base;
              f->_IO_write_ptr = f->_IO_read_ptr;
              f->_IO_write_base = f->_IO_write_ptr;
              f->_IO_write_end = f->_IO_buf_end;
              f->_IO_read_base = f->_IO_read_ptr = f->_IO_read_end;
              f->_flags |= _IO_CURRENTLY_PUTTING;
              if (f->_mode <= 0 && f->_flags & (_IO_LINE_BUF | _IO_UNBUFFERED))
                f->_IO_write_end = f->_IO_write_ptr;
            }
          if (ch == EOF)
            return _IO_do_write (f, f->_IO_write_base,
                                 f->_IO_write_ptr - f->_IO_write_base);
          if (f->_IO_write_ptr == f->_IO_buf_end ) /* Buffer is really full */
            if (_IO_do_flush (f) == EOF)
              return EOF;
          *f->_IO_write_ptr++ = ch;
          if ((f->_flags & _IO_UNBUFFERED)
              || ((f->_flags & _IO_LINE_BUF) && ch == '\n'))
            if (**_IO_do_write (f, f->_IO_write_base,
                              f->_IO_write_ptr - f->_IO_write_base)** == EOF)
              return EOF;
          return (unsigned char) ch;
        }

    - 아래쪽에 노란 부분을 보면, flag에 UNBUFFERED와 LINE_BUF가 설정되어 있고 ch가 개행이면 ```IO_do_write```함수를 호출한다<br><br><br>

6. **IO_do_write 함수**

        int
        **_IO_new_do_write** (FILE *fp, const char *data, size_t to_do)
        {
          return (to_do == 0
                  || (size_t) new_do_write (fp, data, to_do) == to_do) ? 0 : EOF;
        }
        libc_hidden_ver (_IO_new_do_write, _IO_do_write)
        static size_t
        **new_do_write** (FILE *fp, const char *data, size_t to_do)
        {
          size_t count;
          if (fp->_flags & _IO_IS_APPENDING)
            /* On a system without a proper O_APPEND implementation,
               you would need to sys_seek(0, SEEK_END) here, but is
               not needed nor desirable for Unix- or Posix-like systems.
               Instead, just indicate that offset (before and after) is
               unpredictable. */
            fp->_offset = _IO_pos_BAD;
          else if (fp->_IO_read_end != fp->_IO_write_base)
            {
              off64_t new_pos
                = _IO_SYSSEEK (fp, fp->_IO_write_base - fp->_IO_read_end, 1);
              if (new_pos == _IO_pos_BAD)
                return 0;
              fp->_offset = new_pos;
            }
          count = **_IO_SYSWRITE (fp, data, to_do);**
          if (fp->_cur_column && count)
            fp->_cur_column = _IO_adjust_column (fp->_cur_column - 1, data, count) + 1;
          _IO_setg (fp, fp->_IO_buf_base, fp->_IO_buf_base, fp->_IO_buf_base);
          fp->_IO_write_base = fp->_IO_write_ptr = fp->_IO_buf_base;
          fp->_IO_write_end = (fp->_mode <= 0
                               && (fp->_flags & (_IO_LINE_BUF | _IO_UNBUFFERED))
                               ? fp->_IO_buf_base : fp->_IO_buf_end);
          return count;
        }

    - **디버깅1**

        ![]({{ site.baseurl }}/images/study/pwnable/stdout%20file%20structure%20flag%20libc%20leak/Untitled%203.png)

        IO_do_write 함수에 bp를 걸고 실행을 하면 IO_new_do_write함수에 걸린다. 아마 둘이 같은함수인거같다. 해당 함수가 호출되면 rdx가 0이면 ret로 가고 0이 아니면 +16라인으로 jump를 한다. 현재 rdx는 0이므로 ret로 가게 된다<br><br>

    - **디버깅2**

        ![]({{ site.baseurl }}/images/study/pwnable/stdout%20file%20structure%20flag%20libc%20leak/Untitled%204.png)

        ret가 진행되면 해당 함수를 호출했던 IO_new_file_xsputn함수로 돌아간다. ni로 넘기다 보면 rax+0x78에 들어있는 값을 call하게 되는데 해당 함수가 어떤건지 확인해보자<br><br>

    - **디버깅3**

        ![]({{ site.baseurl }}/images/study/pwnable/stdout%20file%20structure%20flag%20libc%20leak/Untitled%205.png)

        ```_IO_new_file_write`` 함수가 호출되는것을 볼수 있다. 해당 함수에서 결국 write함수가 호출되는 것을 볼수 있다. 정상적인 puts의 실행인 경우, 첫번째 인자로 fd인 1이 들어가고 두번째 인자로 문자열이 저장된 주소, 그다음 마지막 인자로 출력할 사이즈가 들어가게 된다<br><br>

        그렇다면 이제 stdout의 flag에 IO_appending을 추가하고, 뒤에 NULL byte를 25바이트 추가했을때의 메모리 상태를 확인해보자.<br><br><br>

7. **stdout flag 변조 및 NULL 25byte 추가**
    - 우선 set 명령어로 flag 변조 및 null 바이트 25개를 추가하였다
        ```md
        gdb-peda$ set {int}0x7ffff7dd2620=4222425088 // 0xfbad3887
        gdb-peda$ set {double}0x7ffff7dd2628=0
        gdb-peda$ set {double}0x7ffff7dd2630=0
        gdb-peda$ set {double}0x7ffff7dd2638=0
        gdb-peda$ set {char}0x7ffff7dd2640=0 // total 25byte null 추가
        ```

    - **디버깅 1** (b* _IO_do_write)

        ![]({{ site.baseurl }}/images/study/pwnable/stdout%20file%20structure%20flag%20libc%20leak/Untitled%206.png)

        set으로 위 주소의 들어있는 값들을 변경하면 `_IO_do_write` 가 실행될때 rdx가 0xa3으로 변경되어있는것을 확인가능하다. 따라서 정상적인 루틴에서는 rdx가 0 이여서 ret로 갔지만 지금은 +16주소로 jump할 것이다.<br><br>

    - **디버깅2**

        ![]({{ site.baseurl }}/images/study/pwnable/stdout%20file%20structure%20flag%20libc%20leak/Untitled%207.png)

        아까처럼 call이 진행된다. 그렇다면 해당 함수는 동일하게 ```_IO_new_file_write``` 함수일 것이다<br><br>

    - **디버깅3**

        ![]({{ site.baseurl }}/images/study/pwnable/stdout%20file%20structure%20flag%20libc%20leak/Untitled%208.png)

        아까처럼 결국 write함수가 호출될텐데, 가지고가는 인자를 확인하면 첫번째 인자는 동일하게 fd=1이지만 두번째 인자는 문자열이 저장된 주소가 아닌, `0x7ffff7dd2600` 의 주소가 들어가 있다. 또한 사이즈 부분도 0xa3이 들어가 있다.

        따라서 `0x7ffff7dd2600`  에 들어있는 값들이 0xa3만큼 출력이 될 것이다.<br><br>

        ![]({{ site.baseurl }}/images/study/pwnable/stdout%20file%20structure%20flag%20libc%20leak/Untitled%209.png)

        해당 주소에는 저런 값들이 들어있고, 빨간색 부분 값을 이용한다면, 해당 주소를 leak한뒤 -131 를 계산해주면 stdout의 주소를 알 수 있다.<br><br>

    - write함수에 저렇게 인자가 들어가는 이유
        1. **_IO_do_write (f, f->_IO_write_base, f->_IO_write_ptr - f->_IO_write_base)**

            아까 _IO_new_file_overflow  함수에서 IO_do_write함수가 호출된다고 했는데 해당 인자를 확인해보자.

            - f는 stdout의 주소가 들어감
            - 두번째는 stdout 구조체의 write_base 멤버 변수 값이 들어감
            - 세번쨰는 write_ptr 멤버변수 값 - write_base 멤버변수 값의 계산된 값이 들어간다

            ![]({{ site.baseurl }}/images/study/pwnable/stdout%20file%20structure%20flag%20libc%20leak/Untitled%2010.png)<br><br>

        2. 따라서 최종적으로 다음과 같이 write함수가 호출된다.

            ```write( 0x7ffff7dd2620, 0x7ffff7dd2600, 0xa3)```

            - 결과

                ![]({{ site.baseurl }}/images/study/pwnable/stdout%20file%20structure%20flag%20libc%20leak/Untitled%2011.png)