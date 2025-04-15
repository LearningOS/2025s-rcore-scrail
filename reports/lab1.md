### 功能总结

添加了`sys_trace`系统调用，用以实现地址读写和系统调用统计功能。读写按照目前的要求比较简单，直接对地址进行指针类型转换并读写即可。对于调用统计，在Task控制块中添加了一个数组用于记录。但是如果直接将长度设为512会导致占用空间过大而使得内核代码的其他部分内容被影响。考虑到用户态代码中提供的全部系统调用不超过50个，这里编写了一个映射函数`map_syscall`将稀疏的调用号映射到连续下标进而减少空间浪费。

### 简答题

#### 正确进入 U 态后，程序的特征还应有：使用 S 态特权指令，访问 S 态寄存器后会报错。 请同学们可以自行测试这些内容（运行 三个 bad 测例 (ch2b_bad_*.rs) ）， 描述程序出错行为，同时注意注明你使用的 sbi 及其版本。

```shell
[rustsbi] RustSBI version 0.3.0-alpha.2, adapting to RISC-V SBI v1.0.0
.______       __    __      _______.___________.  _______..______   __
|   _  \     |  |  |  |    /       |           | /       ||   _  \ |  |
|  |_)  |    |  |  |  |   |   (----`---|  |----`|   (----`|  |_)  ||  |
|      /     |  |  |  |    \   \       |  |      \   \    |   _  < |  |
|  |\  \----.|  `--'  |.----)   |      |  |  .----)   |   |  |_)  ||  |
| _| `._____| \______/ |_______/       |__|  |_______/    |______/ |__|
[rustsbi] Implementation     : RustSBI-QEMU Version 0.2.0-alpha.2
[rustsbi] Platform Name      : riscv-virtio,qemu
[rustsbi] Platform SMP       : 1
[rustsbi] Platform Memory    : 0x80000000..0x88000000
[rustsbi] Boot HART          : 0
[rustsbi] Device Tree Region : 0x87000000..0x87000ef2
[rustsbi] Firmware Address   : 0x80000000
[rustsbi] Supervisor Address : 0x80200000
[rustsbi] pmp01: 0x00000000..0x80000000 (-wr)
[rustsbi] pmp02: 0x80000000..0x80200000 (---)
[rustsbi] pmp03: 0x80200000..0x88000000 (xwr)
[rustsbi] pmp04: 0x88000000..0x00000000 (-wr)
[kernel] Hello, world!
[kernel] PageFault in application, bad addr = 0x0, bad instruction = 0x804003a4, kernel killed it.
[kernel] IllegalInstruction in application, kernel killed it.
[kernel] IllegalInstruction in application, kernel killed it.
[kernel] Panicked at src/task/mod.rs:139 All applications completed!
```
读地址导致缺页，然后被内核杀掉，其余两个由于指令本身不能在用户态权限下执行，直接被杀掉。
#### 深入理解 trap.S 中两个函数 __alltraps 和 __restore 的作用，并回答如下问题:

1. L40：刚进入 __restore 时，sp 代表了什么值。请指出 __restore 的两种使用情景。

有两种进入__restore的场景：
- 在trap后复原并回归用户态
- 系统启动后第一次进行加载

在系统启动后，会执行第一个app程序。在初始化过程中，已经构造好最开始的任务上下文，并在执行app0时调用__switch切换这个特定的上下文，由于ra被指向__restore，因此执行从S态到U态的转化。此时sp指向内核栈。

而在trap后进入__restore，那就是在`call trap_handler`之后，而前文的`__alltraps`保证sp指向内核栈顶，因此sp的含义在两种情况下是一致的。

2. L43-L48：这几行汇编代码特殊处理了哪些寄存器？这些寄存器的的值对于进入用户态有何意义？请分别解释。
```asm
ld t0, 32*8(sp)
ld t1, 33*8(sp)
ld t2, 2*8(sp)
csrw sstatus, t0
csrw sepc, t1
csrw sscratch, t2
```
前三行从内存中恢复trap上下文的这三个寄存器值`t0=sstatus`, `t1=sepc`,`t2=sp`，并写回对应的csr位置。注意到这里`sscratch`保存了`t2`即此时存下了用户态栈顶。

3. L50-L56：为何跳过了 x2 和 x4？

```asm
ld x1, 1*8(sp)
ld x3, 3*8(sp)
.set n, 5
.rept 27
   LOAD_GP %n
   .set n, n+1
.endr
```
`x2`是`sp`在处理完上下文之前不能变。`tp`考虑到其线程指针的含义，似乎并没有用到，因而没有恢复。

4. L60：该指令之后，sp 和 sscratch 中的值分别有什么意义？

```asm
csrrw sp, sscratch, sp
```

`sp`指向用户栈顶，`sscratch`指向内核栈顶。

5. __restore：中发生状态切换在哪一条指令？为何该指令执行之后会进入用户态？

`sret`，由riscv架构保证实现返回用户态。具体而言，这一指令将`pc`设置为`sepc`，复原`sstatus`到trap之前等等。

6. L13：该指令之后，sp 和 sscratch 中的值分别有什么意义？
```asm
csrrw sp, sscratch, sp
```
`sp`保存内核栈顶，`sscratch`保存用户栈顶。

7. 从 U 态进入 S 态是哪一条指令发生的？

在用户态执行`ecall`时发生
### 荣誉准则

1. 在完成本次实验的过程（含此前学习的过程）中，我曾分别与 以下各位 就（与本次实验相关的）以下方面做过交流，还在代码中对应的位置以注释形式记录了具体的交流对象及内容：

无

2. 此外，我也参考了 以下资料 ，还在代码中对应的位置以注释形式记录了具体的参考来源及内容：

《RISC-V 手册》

3. 我独立完成了本次实验除以上方面之外的所有工作，包括代码与文档。 我清楚地知道，从以上方面获得的信息在一定程度上降低了实验难度，可能会影响起评分。

4. 我从未使用过他人的代码，不管是原封不动地复制，还是经过了某些等价转换。 我未曾也不会向他人（含此后各届同学）复制或公开我的实验代码，我有义务妥善保管好它们。 我提交至本实验的评测系统的代码，均无意于破坏或妨碍任何计算机系统的正常运转。 我清楚地知道，以上情况均为本课程纪律所禁止，若违反，对应的实验成绩将按“-100”分计。
