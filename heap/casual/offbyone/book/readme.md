**利用：offbyone溢出。**
程序开启了以下保护：
```
[*] '/mnt/hgfs/Desktop/offbyone/book/b00ks'
    Arch:     amd64-64-little
    RELRO:    Full RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      PIE enabled
```
由于开启了PIE，不好调试。但是本地可以关闭ASLR,那么程序加载的基址是固定的，加上IDA中的偏移即可下断点。加载的基址可以通过查看/proc/xxx/maps得到：
```
555555554000-555555556000 r-xp 00000000 00:32 26189                      /mnt/hgfs/Desktop/offbyone/book/b00ks
555555755000-555555756000 r--p 00001000 00:32 26189                      /mnt/hgfs/Desktop/offbyone/book/b00ks
555555756000-555555757000 rw-p 00002000 00:32 26189                      /mnt/hgfs/Desktop/offbyone/book/b00ks
7ffff7a0d000-7ffff7bcd000 r-xp 00000000 08:01 1050563                    /lib/x86_64-linux-gnu/libc-2.23.so
7ffff7bcd000-7ffff7dcd000 ---p 001c0000 08:01 1050563                    /lib/x86_64-linux-gnu/libc-2.23.so
7ffff7dcd000-7ffff7dd1000 r--p 001c0000 08:01 1050563                    /lib/x86_64-linux-gnu/libc-2.23.so
7ffff7dd1000-7ffff7dd3000 rw-p 001c4000 08:01 1050563                    /lib/x86_64-linux-gnu/libc-2.23.so
7ffff7dd3000-7ffff7dd7000 rw-p 00000000 00:00 0 
7ffff7dd7000-7ffff7dfd000 r-xp 00000000 08:01 1050535                    /lib/x86_64-linux-gnu/ld-2.23.so
7ffff7fd7000-7ffff7fda000 rw-p 00000000 00:00 0 
7ffff7ff7000-7ffff7ffa000 r--p 00000000 00:00 0                          [vvar]
7ffff7ffa000-7ffff7ffc000 r-xp 00000000 00:00 0                          [vdso]
7ffff7ffc000-7ffff7ffd000 r--p 00025000 08:01 1050535                    /lib/x86_64-linux-gnu/ld-2.23.so
7ffff7ffd000-7ffff7ffe000 rw-p 00026000 08:01 1050535                    /lib/x86_64-linux-gnu/ld-2.23.so
7ffff7ffe000-7ffff7fff000 rw-p 00000000 00:00 0 
7ffffffde000-7ffffffff000 rw-p 00000000 00:00 0                          [stack]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]
```
其中0x555555554000即是加载基址。

create函数创建的结构体如下：
```
    struct book_structure
    {
       long long book_rank
       char *name_ptr
       char *description
       long long size
    }
    ----low addr ----
    name chunk
    description chunk
    book structure chunk
    ----high addr-----
```

my_read函数中有NULL字节溢出：
```
signed __int64 __fastcall read_offbyone(_BYTE *a1, int a2)
{
  int i; // [rsp+14h] [rbp-Ch]
  _BYTE *buf; // [rsp+18h] [rbp-8h]
  if ( a2 <= 0 )
    return 0LL;
  buf = a1;
  for ( i = 0; ; ++i )
  {
    if ( (unsigned int)read(0, buf, 1uLL) != 1 )
      return 1LL;
    if ( *buf == 10 ) //如果是回车，直接break，将回车替换成\x00
      break;
    ++buf;
    if ( i == a2 )
      break;
  }
  *buf = 0;
  return 0LL;
}
```
**输入字符串后会自动在字符串的末尾加上\x00来截断字符串。**

程序开始时输入author名字：
```
signed __int64 change()
{
  printf("Enter author name: ");
  if ( !(unsigned int)read_offbyone(authorname_ptr, 32) )
    return 0LL;
  printf("fail to read author_name", 32LL);
  return 1LL;
}
```
如果输入的字符串的长度是32，那么\X00字节就会加在第33个字节上,即加上book_struct中，这时create一个book，则会将这个\x00覆盖掉。
```
.data:0000000000202010 book_struct_ptr dq offset book_struct   ; DATA XREF: sub_B24:loc_B38↑o
.data:0000000000202010                                         ; delete:loc_C1B↑o ...
.data:0000000000202018 authorname_ptr  dq offset author_name   ; DATA XREF: change+15↑o

.bss:0000000000202040 author_name     db 20h dup(?)           ; DATA XREF: .data:authorname_ptr↑o
.bss:0000000000202060 book_struct     db    ? ;               ; DATA XREF: .data:book_struct_ptr↑o
```