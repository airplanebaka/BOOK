---
description: 一些环境搭建说明
---

# 第一章 环境搭建

## 通用环境搭建说明

本工程在windows10操作系统下进行，涉及到的软件如 Quartus 18.0\(prime\), modelsim 等均在windows下使用

> modelsim在Quartus安装的时候可以勾选一并安装

对于其中涉及到linux的相关操作，全部在WSL\(linux子系统ubuntu16.04进行\)，当然也可以使用虚拟机操作，不过这样速度会慢一些。需要注意的是WSL默认挂载在C盘，如果C盘空间不足，请更改挂载位置，由于本人只有C盘是固态，所以按照默认方式进行。

> 在WLS下可以通过 `cd /mnt/c/...`的方式，迅速的切换到C盘目录\(其他盘类推\)，之后便可以用命令行的形式在windows下工作，由于本人比较熟悉命令行与vim，所以经常使用此方法。

### RISCV交叉编译链

#### 安装命令

```text
$ sudo apt-get install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev
$ git clone --recursive https://github.com/riscv/riscv-gnu-toolchain
$ cd riscv-gnu-toolchain
$ ./configure --prefix=/opt/riscv --with-arch=rv32ima --with-abi=ilp32
$ make -j4
```

> arch中的a参数代表原子指令，对于跑操作系统来讲是不可或缺的，另外由于我选用的cpu没有fpu硬件，所以采用ilp32软浮点模拟的功能
>
> 由于编译链比较大，且可能会访问一些google源，所以请确保网络通畅，整个 `git clone update` 过程大约需要持续数小时，编译也是需要数小时，请耐心等待

更详细的参数请参考 [https://github.com/riscv/riscv-gnu-toolchain](https://github.com/riscv/riscv-gnu-toolchain) 的说明

### 添加环境变量

在 `~/.bashrc`中添加 `PATH=/opt/riscv/bin:$PATH` 之后键入

```text
$ source ~/.bashrc
```

接下来在任何一个地方命令行输入 riscv32 后输入两次 Tab, 应该会有自动补全成 **riscv32-unknown-elf-** 并显示若干编译链工具，至此编译链安装成功。

如果安装了qemu，可以交叉一个riscv32格式的helloworld，并用qemu-riscv32运行之，用以简单的测试。

### 安装rust编译链

#### 安装命令:

```text
$ curl https://sh.rustup.rs -sSf | sh
$ source $HOME/.cargo/env
$ rustup update
```

此时输入 `rustc --version` 应该会有类似于`rustc x.y.z (abcabcabc yyyy-mm-dd)`这样的信息, 接下来执行

```text
$ cargo install cargo-xbuild
```

至此安装完毕

### 运行本工程以及分支介绍

好了，这个时候假定你已经正确地安装好了编译链，下面开始在modelsim下仿真本项目:

> 由于网速原因，有的时候会很慢，所以可以 git clone -b xxx -depth=1 指定分支clone，一般一个分支不会大于100MB \(TODO 整理github减少一些不必要的文件\)

```text
git clone git@github.com:oscourse-tsinghua/undergraduate-fzpeng2020.git
git checkout feature-4MB-bbl-without-compression
./build.sh
```

> **虚拟机注意** 可能在运行./build.sh中遇到dummpy\_payload下的dummpy\_sbi.S出现问题，这是由于共享文件夹引起软链接失效导致的，在WSL上则不会有问题 \(此处感谢贺清同学\)
>
> 注意涉及OS部分操作目前仅支持在仿真环境下运行

好了，接下来打开quartus,由于每个人的modelsim路径不一样，第一次需要设置modelsim的路径 `Tools->Options->General->EDA Tool Options`下设置相应的路径:

> 需要注意的是quartus和modelsim的关联设置，这里默认的是modelsim-altera，如是SE版本请自行在Assignments-&gt;Setting-&gt;EDA TOOL Settings-&gt;Simulation-&gt;Tool Name下选择合适的仿真版本。 如果改成SE版本需要在 `Assignments->Settings->EDA Tool Settings`下更改Tool Name

然后点击`Tool->Run Simulation Tool->RTL Simualtion` 不出问题的话，就应该可以自动跳转到modelsim仿真界面了。

**说明： 一般分支下有两个quartus project分别叫wishbone\_cyc10, wishbone\_cyc10\_os,前者测指令，后者测OS**

#### 分支: feature-4MB-bbl-without-compression

这个分支对应于仿真环境下\(modelsim\)可以跑操作系统,只需看**wishbone\_cyc10\_os**这个工程即可。对于这个分支而言需要先运行`build.sh`这样就可以正常仿真了。

#### 分支：master

这个分支下只需要考虑工程**wishbone\_cyc10**即可, 用的是第三方SDRAM IP核，可以下板\(SDRAM有8MB,但实际上只有部分空间可以使用\)。

#### 分支: sdram\_qsys

这个分支下只需考虑工程**wishbone\_cyc10**即可，用的是quartus自带的IP核\(和master的不一样\), sdram仍有bug。

#### 分支: sdram\_naked

无CPU，只是用来测试SDRAM的，里面有一个可以认为是信号发生器的master，逻辑简单，可以用来验证SDRAM\(虽然bug未解决\)。

## Chisel环境相关

### 安装SBT工具

进入官网下载

* [https://www.scala-sbt.org/download.html](https://www.scala-sbt.org/download.html)
* 下载最新sbt-x.x.x.tgz安装包

将下载的安装包移动到安装java包的目录

```scala
// 解压安装包
tar zxvf sbt-x.x.x.tgz

// 在安装目录创建sbt文件
vim sbt

// 在sbt中添加如下代码
#!/bin/bash
BT_OPTS="-Xms2048M -Xmx4096M -Xss1M -XX:+CMSClassUnloadingEnabled -XX:MaxPermSize=256M"
java $SBT_OPTS -jar /home/soft/sbt/bin/sbt-launch.jar "$@"
// *注意更改/home/soft/为你的安装目录*
 
// 修改sbt文件权限
chmod u+x sbt

// 配置sbt环境变量
vim /etc/profile
// *在末尾添加*
export SBT_HOME=/home/soft/sbt
export PATH=${SBT_HOME}/bin:$PATH
// *注意更改/home/soft/为你的安装目录*

// 生效配置
source /etc/profile

// 检测是否安装成功
sbt -version

```

### 安装Chisel开发软件

安装前说明：Java一定要安装成功

#### intellij下载

* [https://www.jetbrains.com/idea/download](https://www.jetbrains.com/idea/download)

```scala
// 安装intellij
// 在sudo环境下安装
tar -xf ideaIU-2017.3.4-no-jdk.tar.gz
mv idea-IU-173.4548.28 /opt
cd /opt/idea-IU-173.4548.28/bin

// 运行intellij
./idea.sh

// *注意修改自己的版本号和安装地址*
```

#### 安装Scala支持

随意建立一个工程，创建完毕后选择File -&gt; Setting，选择选项卡中的Plugins，搜索Scala，安装人气最高的那个插件，等安装完后重启intelij。

请自行测试是否能运行scala文件，既编写hello world测试。

#### 安装Chisel支持

![](.gitbook/assets/image%20%2815%29.png)

![](.gitbook/assets/image%20%284%29.png)

```scala
// 打开左侧project中的build.sbt
// 替换为如下代码
def scalacOptionsVersion(scalaVersion: String): Seq[String] = {
  Seq() ++ {
    // If we're building with Scala > 2.11, enable the compile option
    //  switch to support our anonymous Bundle definitions:
    //  https://github.com/scala/bug/issues/10047
    CrossVersion.partialVersion(scalaVersion) match {
      case Some((2, scalaMajor: Long)) if scalaMajor < 12 => Seq()
      case _ => Seq("-Xsource:2.11")
    }
  }
}

def javacOptionsVersion(scalaVersion: String): Seq[String] = {
  Seq() ++ {
    // Scala 2.12 requires Java 8. We continue to generate
    //  Java 7 compatible code for Scala 2.11
    //  for compatibility with old clients.
    CrossVersion.partialVersion(scalaVersion) match {
      case Some((2, scalaMajor: Long)) if scalaMajor < 12 =>
        Seq("-source", "1.7", "-target", "1.7")
      case _ =>
        Seq("-source", "1.8", "-target", "1.8")
    }
  }
}


name := "c_t_1" // 工程名自行更改

version := "0.1" // 版本参考未替换前

scalaVersion := "2.11.12" // 参考自己安装的Scala版本

crossScalaVersions := Seq("2.11.12", "2.12.4")

resolvers ++= Seq(
  Resolver.sonatypeRepo("snapshots"),
  Resolver.sonatypeRepo("releases")
)

// Provide a managed dependency on X if -DXVersion="" is supplied on the command line.
val defaultVersions = Map(
  "chisel3" -> "3.2-SNAPSHOT",
  "chisel-iotesters" -> "1.3-SNAPSHOT"
)

libraryDependencies ++= Seq("chisel3","chisel-iotesters").map {
  dep: String => "edu.berkeley.cs" %% dep % sys.props.getOrElse(dep + "Version", defaultVersions(dep)) }

scalacOptions ++= scalacOptionsVersion(scalaVersion.value)

javacOptions ++= javacOptionsVersion(scalaVersion.value)

// 配置完后保存，软件自动开始下载所需要的依赖
// 下载依赖需要访问外放，请自动选择相关操作
```

依赖和配置成功后就可以编写Chisel代码了

### IDEA部分错误解决方法

#### 图形界面问题

```scala
// 错误代码
can’t connect to x11 window server using ‘localhost :10.0’ as value of the DISPLAY variable.
```

出现上述错误的原因为windows内核集成了gui，所以本地可以展示验证码，而linux上没有启动x server，故没有展示。

{% hint style="info" %}
解决方法

1.重启x server

2.在运行项目时加上-Djava.awt.headless=true

3.如果是在root环境下启动失败则退出root在用户模式下启动

4.如果是在用户模式下启动失败则进入root在该模式下启动
{% endhint %}

## Quartus安装

下载Quartus安装包

* [https://fpgasoftware.intel.com/?edition=lite](https://fpgasoftware.intel.com/?edition=lite)

![](.gitbook/assets/image%20%283%29.png)

![](.gitbook/assets/image%20%2818%29.png)

下载安装完成后即可打开代码仓库中的Quartus工程

