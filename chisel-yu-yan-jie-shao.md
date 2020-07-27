# 第三章 Chisel语言相关

## Chisel简介

        在硬件设计上大家更加了解Verilog这门HDL语言。我们都知道Verilog语言太啰嗦，一个简单模块可能要写满屏幕的代码，导致可读性和维护性都很差。另一个大问题就是参数化很差，参数传递不清楚，封装不干净，导致代码量过大和复用性很低。

        其他方面将其他语言转换为Verilog，例如C到Verilog。但是这种转换效率低下，设计者必须学习电路知识，学习工具规范后才能开发，而且写硬件的语法还和写软件不一样，导致许多奇奇怪怪的错误，甚至转换后的代码也存在诸多BUG。这些问题将高级语言的高效性除去，增加了繁琐的操作，大大降低了效率。

        而一种语言打破了这种限制，那就是Chisel。 Chisel（Constructing Hardware In a Scala Embedded Language）是UC Berkeley开发的一种开源硬件构造语言。它是建构在Scala语言之上的领域专用语言（DSL），支持高度参数化的硬件生成器。Chisel是基于Scala的语言，所以它继承了便于封装和扩展的特性。chisel设计了bundle的概念，于是成千上百的端口互联只用一句程序就可以实现，干净简单还不容易出错。在编译器方面，chisel的编译器可以实现功能相当强大的错误检查。基本上，如果一个reg或者wire是中间变量，chisel不建议我们声明它的位宽，编译器会在编译过程中自动推断它的位宽。在一个连接链上，编译器会检查所有的东西位宽一致。这就杜绝了很多低级人为错误的出现。

        由于Chisel的学习具有较高的门槛，要求大家熟练掌握对面向对象编程，导致最近几年Chisel的热度下降，只有部分高校和公司在使用。

## 模块替换步骤

{% hint style="info" %}
编译使用软件为Linux Ubuntu系统中的intellij，根据第一章安装。

Chisel编译后会存在always中时序与原来不相符，根据自己设定的时钟名称修改编译后默认clock名称。
{% endhint %}

        使用软件打开工程，src/main/scala/CPU/下为主要文件

```scala
Main.scala文件为编译文件，将各个模块编译成Verilog语言代码
package CPU

object Main extends App{
  chisel3.Driver.execute(args, () => new CSR)// 根据需求更改
}
```

## 模块替换部分

### 译码模块

```scala
def default = List(d.EXE_NOP_OP, d.EXE_RES_NOP, d.Y,// aluop alusel insval
                     d.WriteDisable, d.ReadDisable, d.ReadDisable,// regw reg1 reg2
                     d.ZeroWord,// imm
                     d.ZeroWord,// mem_addr
                     d.ZeroWord, d.X, d.NotInDelaySlot, d.X,// jaddr jflag delay step
                     d.X, d.X, d.X, d.X, d.X,// sys bre sret mret fence
                     d.ReadDisable, d.CSRWriteDisable)// csr_read_en csr_write_en
```

        IDecode模块主要部分为上面所示的指令信号列表。

```scala
def what_you_want         = BitPat("b????????????????????????????????")
...
val table = Array(
   ins.what_you_want ->
     List(d.EXE_ADD_OP, d.EXE_RES_ARITHMETIC, d.X,
      d.WriteEnable, d.ReadEnable, d.ReadDisable,
      imm_i_type,
      d.ZeroWord,
      d.ZeroWord, d.X, d.NotInDelaySlot, d.X,
      d.X, d.X, d.X, d.X, d.X,
      d.ReadDisable, d.CSRWriteDisable)
    )
```

        新增指令需要添加定义对应的BitPat码，然后按照信号的设定对List中相应位置进行修改。如果存在未被定义的信号量，请在default中进行添加，然后在对应指令List中末尾进行设置。

### 存储器模块

        存储器模块IMemory逻辑与原有的Verilog语言mem模块逻辑相同。存在差异的地方为需要使用when条件语句代替原有的case语句，因为Scala所使用的switch语句并没有default语法，而模块中需要对条件未命中时进行设定，所以建议采用when—otherwise语句。

```scala
// Verilog中mem模块部分
// LB指令相关
case(mem_addr_i[1:0])
							xxx：xxx
							default:
							begin
								wdata_o <= `ZeroWord;
								mem_sel_o <= 4'b0000;
								mem_ce <= `ChipDisable;
								load_alignment_error <= `True_v;
							end
						endcase

// Chisel中LB相关描述
when(io.mem_addr_i(1,0) === "b00".U(2.W)){
           xxxx
					 
          }.otherwise{
            wdata_o := d.ZeroWord
            mem_sel_o := "b0000".U(4.W)
            mem_ce_o := d.ChipDisable
            load_alignment_error := true.B
          }
```

### 其他模块说明

        其他模块的逻辑设计并没和原Verilog语言设计的逻辑有很大差别，所以在这并不多说。

        在所有模块中通用的一种修改为关于subword\(x\)这种语法。

```scala
// 在Verilog中实现这种语法非常的常见
// 例如
reg[31：0] excepttype;
always @(*) begin
    excepttype(0) <= 1'h0;
    excepttype(1) <= 1'h0;
    ...
    excepttype(31) <= 1'h0;
end
// 在verilog中能够快速的对多位变量中的某一位进行修改
```

        而在Chisel中这种对subword的设定是非法的，编译会产生错误。

```scala
// 以下代码出于官方解释
// Example:
class TestModule extends Module {
  val io = IO(new Bundle {
    val in = Input(UInt(10.W))
    val bit = Input(Bool())
    val out = Output(UInt(10.W))
  })
  io.out(0) := io.bit
}
// chisel并不支持subword赋值；但是使用Bundles和Vec能够好的表达这种类型语句
// 官方给出的修改
class TestModule extends Module {
  val io = IO(new Bundle {
    val in = Input(UInt(10.W))
    val bit = Input(Bool())
    val out = Output(UInt(10.W))
  })
  val bools = VecInit(io.in.toBools)
  bools(0) := io.bit
  io.out := bools.asUInt
}
```

        按照官方wiki对subword的解决方法，可以参考修改excepttype

```scala
val excepttype = VecInit(d.ZeroWord.asBools)
    excepttype(0) := false.b
    excepttype(1) := false.b
    ...
    excepttype(31) := false.b
    io.excepttype := excepttype.asUInt
```

## 问题汇总

{% hint style="info" %}
首先有问题请先看Chisel的wiki，[https://github.com/freechipsproject/chisel3/wiki](https://github.com/freechipsproject/chisel3/wiki)
{% endhint %}

### 初始化问题

在编译的时候，编译器会弹出XXX变量未初始化的问题

```scala
// 初始化问题
// 当一个变量未被赋予初值时并在语句内被使用，就会产生未初始化错误
// 例如

when(...){
    x1 := 0.U(32.W)
}
// 编译后会错误
// 修改
val x1 = Reg(UInt(32.W))
x1 ：= 0.U(32.W)
when(...){
    x1 := 0.U(32.W)
}
```

### 组合逻辑循环错误

同样是编译时候发生的错误，由FIRRTL中间过程产生，错误为Combinational logic loop error

```scala
// 产生组合逻辑循环错误例子
val x1 = Reg(UInt(32.W))
x1 ：= 0.U(32.W)
when(...){
    x1 := x1 + 1.U(32.W)// 此处问题
}
// 此处用于PC模块中PC+4
// 使用Reg类型变量进行操作就会产生错误
// 建议改为Wire类型
```

