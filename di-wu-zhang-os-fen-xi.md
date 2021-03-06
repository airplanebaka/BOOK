# 第六章 OS分析

## OS

这个rCore操作系统已经有十分详细的记录了,可以直接去看这个连接[rCore](https://rcore-os.github.io/rCore_tutorial_doc/)

下面是介绍关于64位到32位移植的操作：

### 页表修改

将`./boot/entry.asm`的三级映射改成二级映射, 一开始是一个粗糙的映射，实际上需要说明的是，这个映射不做也是可以的，因为bbl已经帮我们设置了，这里设置的一个目的在于兼容qemu的启动过程，如果不设置的话，用qemu模拟仿真可能会出现问题。

```text
boot_page_table_sv32:
    .zero 4 * 513
    # 0x80400000 -> 0x80400000 (4M)
    .word 0x201000cf # VRWXAD
    .zero 4 * 255
    # 0xC0400000 -> 0x80400000 (4M)
    .word 0x201000ef #DAG XWRV
    .zero 4 * 253
    # sbi map
    .word 0x20108401 #DA     V
boot_page_table_sv32_top:
```

实际上这部分如法炮制，照葫芦画瓢也很容易，我这里最后把刷tlb给注释了，也就是说这个设置其实是不必须的，当然跑qemu的时候还是设置上比较好。

```text
    lui     t0, %hi(boot_page_table_sv32)
    li      t1, 0xC0000000 - 0x80000000
    sub     t0, t0, t1
    srli    t0, t0, 12
    li      t1, 1 << 31
    or      t0, t0, t1
    csrw    satp, t0
    #sfence.vma
```

此外还需修改TLB策略为RV32，这部分只需修改把所有的RV39替换成RV32就可以，因为第三方库已经写好了\(清华佬tql\)，然后把刷tlb的函数改成32位的:

```text
pub fn token(&self) -> usize { self.root_frame.number() | (1 << 31) }
```

### 寄存器长度修改

这块还需要修改的一个点是`./trap/trap.asm`

```text
.equ XLENB, 4
```

这也很容易，就是改个长度就行，由于都用宏控制了,改一处就可以了。

总的来说OS改动比较少，但关键是理解这个OS，虽然可能只有几行代码但是不了解背后机制可能也无从改起了。

