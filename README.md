# 2019 小学期硬件类实验实验报告 —— 计算机组成原理课程设计实验报告

> 团队项目 —— 流水线 CPU

# 一. 设计目的

## 1.1 2019 小学期硬件类设计实验目的

- 深入理解计算机相关理论
- 熟练掌握计算机软硬件工作原理
- 学习并掌握计算机接口设计
- 学习并掌握 RISC 指令集的处理器设计
- 熟悉 HDL 语言、EDA 工具应用
- 培养计算机系统能力

## 1.2 计算机组成原理课程设计小组项目实验目的

每组同学共同完成一个多周期的或者流水线的 CPU，仿真并在精工板上验证。我们小组选择实现基于 RISC 架构的 MIPS 经典五级流水线 CPU。

# 二. 使用的设备器材

本次实验涉及到的具体设备与器材如下表格所示：

| 设备、器材         | 名称               | 版本            |
| ------------------ | ------------------ | --------------- |
| 操作系统           | Windows 10         | 1903            |
| EDA                | Vivado             | 2017.2          |
| Verilog 代码编辑器 | Visual Studio Code | Version 1.37    |
| MIPS 模拟器        | MARS               | 4.5             |
| 精工开发板         |                    | xc7a35tcsg324-1 |

# 三. 设计原理及步骤

## 3.1 经典五级流水线 CPU 准备实现的指令

我们小组经过商讨，考虑到指令集的完备性与完整性，最终决定实现的 MIPS 指令集子集包括：

- I 型指令：ADDI、ADDIU、SLTIU、SLTI、ANDI、ORI、XORI、LUI、LW、SW、BEQ、BNE
- R 型指令：ADD、ADDU、SUB、SUBU、SLT、SLTU、AND、OR、XOR、NOR、SLL、SRL、SRA、SLLV、SRLV、SRAV、JR、JALR
- J 型指令：J、JAL

这里面包含了全部的移位指令、全部的 J 型跳转指令与几乎大部分算术运算指令。

## 3.2 流水线 CPU 的基本实现原理

流水线 CPU 顾名思义，就是以流水线的方式将 CPU 需要执行的指令进行顺序执行。LW 指令是一个非常典型的能够将 CPU 指令执行阶段进行划分的指令，其具体指令描述如下：

![](https://i.imgur.com/KrLKxOV.png)

以 LW 指令为例子，我们将 CPU 的指令划分为这样的五个基本执行阶段（Stage）：

- IF - Instruction Fetch：取指令
- ID - Instruction Decode：指令译码
- EX - Execute：指令执行
- MEM - Memory：访存
- WB - Writeback：结果写回

那么对于 LW 来说，这五大阶段分别需要这样的工作：

- IF：读取指令存储器 IM，获得指令
- ID：对获取的指令进行译码，得知指令为 LW
- EX：通过 ALU 计算得知“取存储器”的目标地址
- MEM：读取数据存储器的目标地址，获得对应数据
- WB：将取到的数据写回到目标寄存器

五级流水线 CPU 就是将指令按照这样的五个基本阶段进行划分，在一个指令在某一个阶段执行时，第二条指令同时进入上一个阶段，这样让各个阶段叠加处理指令，形成流水线，从而提升 CPU 整体的吞吐率。

![](https://i.imgur.com/QHFiYLq.png)

流水线 CPU 的提高整体吞吐量的主要方法就是：“让多条指令同时执行”。因此，我们需要在同一个时钟周期内进行多项操作，比如：

- 同时让 PC 自增、与计算两个寄存器之和
- 在取指令的同时执行另外一条指令的读写操作

等等。因此，对于“流水线 CPU”来说，我们也需要复制硬件元件，来让同一个时钟周期中需要多次使用到的硬件能够同时运转。我们设计加入“流水线寄存器”用来控制整个数据通路中不同流水阶段里信号、数据的传递。五级流水分为 IF、ID、EX、MEM 和 WB 五个阶段，我们分别设置 IF/ID、ID/EX、EX/MEM、MEM/WB 这四个流水线寄存器，用来连接流水的五个阶段。

如果不考虑接下来的异常处理，我们大致的流水线 CPU 的数据通路就如下图 2 所示：

![](https://i.imgur.com/RQdpAlI.png)

很容易发现，我们流水线 CPU 中的四个流水寄存器需要将指令处理的数据以及指令的控制信号一级一级的进行前递，比如 LW 指令需要将数据存储器中的数据写入目标寄存器中，目标寄存器最终写入的数据是由指令在第四阶段 MEM 才获取到的，而在第五阶段 WB 才写回，因此，rd（目标寄存器）必须经由全部四个流水线寄存器前递。

同理，控制单元 Control Unit 产生的控制信号也需要经由流水寄存器进行前递。事实上，我们完全可以将控制信号按照指令处理的阶段进行分类，如下表格 2 所示：

![](https://i.imgur.com/sxgN9aV.png)

这样我们就也可以将控制信号一同进行前递。

## 3.3 流水线 CPU 可能出现的冲突与异常

然而，事实上，由于我们在流水线 CPU 运行的过程中会出现“同时执行多条指令”的过程，因此我们无法避免数据相关、控制信号冲突等问题，这些问题我们统称为 Hazards。
在设计过程中，我们需要处理的冲突总共有大致三类：

- 结构冲突：Structural Hazard
- 数据冒险：Data Hazard，主要包含：
    - 数据冲突
    - 访存冲突
- 控制冲突：Control Hazard

接下来的步骤里面，我们就需要对这些数据相关与数据冲突进行处理。

# 四. 冲突处理与代码实现

## 4.1 数据相关与冲突的处理

数据相关主要出现在相邻的或执行时间紧邻的几条指令中，后面的指令用到的操作数或要访问的数据受先执行的指令的影响，但由于在流水线 CPU 中，前面执行的指令还没来得及将数据更新，后面的指令就已经到了取数阶段。或是由于指令跳转，但在跳转执行前就已经取得了跳转指令下面的指令，导致会执行跳转下面的不该执行的指令，可以理解为跳转出现了延时性。以下就为了解决这类问题而进行讨论。

### 4.1.1 结构冲突的解决

我们解决结构冲突的基本方法是：

- 通过改变数据通路，或
- 增加硬件来解决

“结构冲突”在所有的数据冲突中属于相对比较方便解决的一种数据相关问题。这里，我们小组以跳转指令 BEQ 为例子，介绍一下我们是如何解决“结构冲突”的。

首先，BEQ 指令的具体执行如下：如果寄存器 rs 的值等于寄存器 rt 的值则转移，否则顺序执行。转移目标由立即数 offset 左移 2 位并进行有符号扩展的值加上该分支指令对应的延迟槽指令的 PC 计算得到。

在 BEQ 指令中，由于我们需要计算寄存器 rs 与寄存器 rt 的差值，因此事实上这部分需要 ALU 的运算结果，但是由于 BEQ 需要控制单元在译码过程（Instruction Decode）就给出相应的结果，那么我们是来不及将 rs 和 rt 寄存器的值前递给 ALU 再进行计算的，因此我们小组在 ID 阶段单独增加了一个硬件单元 Branch Judge，用来专门判断 rs 和 rt 寄存器的值，以决定 NPCOp 的具体信号。

<img src="https://i.imgur.com/VGLVexb.png" width="60%" style="margin: 20px auto;"/>

单独新增的 Branch Judge 硬件具体实现如下：

```verilog
module branch_judge(
           input  wire[31:0] reg1_data,
           input  wire[31:0] reg2_data,

           output wire       zero       // Calculate whether rs - rt is zero
       );

// rs - rt = diff
wire[32:0] diff;

assign diff = {reg1_data[31], reg1_data} - {reg2_data[31], reg2_data};
assign zero = (diff == 0) ? `BRANCH_TRUE : `BRANCH_FALSE;
endmodule
```

这样，我们就通过增加硬件的方法解决了结构冲突的问题。

### 4.1.2 数据冒险——Data Hazard

数据冒险主要是数据冲突和访存冲突，主要是数据在未来得及更新前就已经被需要了，针对数据来源将其分为数据冲突和访存冲突。数据冲突是数据在运算更新前被使用，访存冲突是数据在被从数据存储器中取出来之前就被使用。

#### 4.1.2.1 数据冲突的解决

对于数据冲突，比如：

```
sub $2, $1, $3
and $12, $2, $5
or $13, $6, $2
add $14, $2, $2
sw $15, 100($2)
```

可以看到，SUB 指令的结果是 AND 指令的输入，也是 OR 指令的输入；而 ADD 指令与 SW 指令的输入同样依赖 SUB 指令。

为了解决此类问题，我们采用数据前递，事实上，对 SUB 指令来说，其结果的产生是在其第三流水阶段：EX，也就是整个流水线 CPU 的第三个时钟周期。而对 SUB 之后的 AND 以及 OR 来说，寄存器 $2 的值是在它们的第三阶段 EX 需要的，也就是流水线 CPU 的第 4-5 个时钟周期。纵观整个流水过程，事实上 $2 的值是在第 3 个时钟周期产生，并在第 4-5 个时钟周期需要，因此我们只需要越过流水线五个阶段中 WB（WriteBack）的过程，让第 3 个时钟周期里面计算产生的 $2 寄存器值 直接赋值 给第 4-5 个时钟周期指令 ADD 以及 OR 指令即可。这种避免「Data Hazard」的方法叫做：数据前递（Data Forwarding）。

我们新增一个硬件元件：Forwarding Unit（前递组件）专门用来处理数据的前递，在出现 Data Hazard 的时候将 ALU 的输出赋予给正确的输入。下图中的 ForwardA 与 ForwardB 就是一个简单的例子：

但解决此类问题的关键是判断出现数据冲突的情况，只有找到数据冲突的地方，才能通过数据前递进行解决。

首先，EX/MEM Data Hazard 会出现在：

- 当前指令在 EX 阶段，且：
- 上一条指令会写入寄存器堆，且：
- 上一条指令的写入地址是当前指令 EX 阶段中 ALU 输入寄存器的一个

```clike
// ALU 第一个操作数
if (EX/MEM.RegWrite = 1 &&
    EX/MEM.RegisterRd == ID/EX.RegisterRs) {
    ForwardA = 2
}

// ALU 第二个操作数
if (EX/MEM.RegWrite = 1 &&
    ID/EX.RegisterRd == ID/EX.RegisterRt) {
    ForwardB = 2
}
```

第二种会出现的 Data Hazard 就是 MEM/WB 类型的 Data Hazard.
MEM/WB Data Hazard 会出现在当前指令处于 EX 阶段，而两个时钟周期之前的指令将同一个寄存器更新了。

```clike
// ALU 第一个操作数
if (MEM/WB.RegWrite == 1 &&
    MEM/WB.RegisterRd == ID/EX.RegisterRs &&
    (EX/MEM.RegisterRd != ID/EX.RegisterRs || EX/MEM.RegWrite == 0)) {
    ForwardA = 1
}

// ALU 第二个操作数
if (MEM/WB.RegWrite == 1 &&
    MEM/WB.RegisterRd == ID/EX.RegisterRt &&
    (EX/MEM.RegisterRd != ID/EX.RegisterRt || EX/MEM.RegWrite == 0)) {
    ForwardB = 1
}
```

在有了冲突检测后我们就可以增加前递单元了，当检测到数据冲突，就通过前递单元进行数据前递，解决数据冲突，其他情况仍正常进行。

#### 4.1.2.2 访存冲突的解决

增加了 Forwarding Unit 的五级流水 CPU 已经能够处理算术运算中涉及到的 Data Hazard，但是对于 LW、SW 等涉及到存储、获取数据存储器中「字」的数据，还是会有尚未解决的问题。
我们并不能规避类似下面的指令带来的数据冲突：

```
lw $2, 20($3)
and $12, $2, $5
```

事实上，在 LW 指令的 MEM 阶段，我们就获得了相应的数据，那么，对于下一条 AND 指令，我们只需要在其 EX 阶段前将流水线 Stall 住一个时钟周期，即可将 Data Memory 的数据前递至正确的地方。

为此我们引入 Stall Unit，用于控制各个缓冲器，来决定五级流水在哪些阶段进行暂停，哪些阶段继续。对于访存冲突的冲突检测方法如下：

```
stall_signal = ((en_reg_write_mem && en_lw_mem) && (dst_reg_mem == rs)) ? `MEM_REGW :
       ((en_reg_write_mem && en_lw_mem) && (dst_reg_mem == rt)) ? `MEM_REGW :
       ((en_reg_write_exe && en_lw_exe) && (dst_reg_exe == rs)) ? `EXE_REGW :
       ((en_reg_write_exe && en_lw_exe) && (dst_reg_exe == rt)) ? `EXE_REGW :
       `NON_REGW;
```

判断是在访存阶段还是执行阶段出现访存冲突，主要是判断写使能和地址是否出现了访存冲突，以此确定哪些阶段发出空指令，哪些阶段维持原参数不变，哪些继续流水。
同时在译码到执行阶段的缓冲器里加入一个状态机，又来取决于出现了访存冲突时获取的数据是原来的数据还是更新成流水后的新数据。

```
if (stall_C[2] == 1 && stall_C[3] == 1 && lw_mem_addr == rs_in) begin
        tempdata_1 <= lw_stall_data;
        temp_state <= 2'b01;
    end
    if (stall_C[2] == 1 && stall_C[3] == 1 && lw_mem_addr == rt_in) begin
        tempdata_2 <= lw_stall_data;
        temp_state <= 2'b10;
    end
```

凸凸凸凸凸凸凸凸凸凸凸凸凸

### 4.1.3 控制冲突的解决

在处理指令的跳转时，我们对「是否跳转」以及「跳转目标」的判断大多都是在 EX 阶段进行的：

- 计算跳转目标地址
- 比较源寄存器的大小关系以及计算 Zero 控制信号

因此对跳转的判断大多情况下都需要在 EX 阶段才能得出结果。但是，我们在流水线 CPU 中需要知道下一条指令取哪一条，这样流水线才能顺利的顺序执行。这种情况下，我们就会遇到 Control Hazard 的问题。

再参考了现实中的常见实现过程，我们决定在指令编辑的过程中在跳转指令下面加入空指令来解决控制冲突。特别地，我们选择使用：

```
sll $0, $0, 0
```

这一指令作为空指令进行处理，这条指令编译为机器码的十六进制表示形式为 `0x00000000`。

## 4.2 数据通路的设计

![](https://i.imgur.com/Hutf9E8.png)

在最终的五级流水 CPU 设计中，我们在原有的 CPU 主要部件的基础之上，增加了：

- 专用于控制流水的：
    - IF/ID 流水寄存器
    - ID/EX 流水寄存器
    - EX/MEM 流水寄存器
    - MEM/WB 流水寄存器
- 专用于判断是否进行跳转（BEQ、BNE 指令）的：Branch Judge 元件
- 专用于处理数据相关问题的：
    - Forwarding Unit：前递单元
    - Stall Unit：优化暂停单元

同时，我们也新增了流水相关的控制信号、前递单元相关信号、优化暂停单元相关信号等等。

特别地，Forwarding Unit 前递单元位于 EX 执行阶段，根据写信号 MEM.RegWrite、WB.RegWrite 以及写寄存器地址、rs 寄存器地址和 rt 寄存器地址判断是否需要进行数据前递，进行 Data Hazard 的解决。

Stall Unit 优化暂停单元控制各个流水寄存器的流通，保证 LW、SW 等读写寄存器指令能够完整执行结束再执行后续指令，以保证访存冲突的解决。
 
## 4.3 Verilog 代码的具体实现

根据数据通路的设计，我们实现了如下的代码结构：

![](https://i.loli.net/2019/09/18/1k6CmZP4QyesfqR.png)

# 五. 仿真与测试

仿真测试部分我们采用了斐波那契数列测试代码：


在流水线 CPU 中，该测试代码的预期结果应为从数字 1 开始依次递增的 1、1、2、3、5、8、13、21、34…… 数列。在实际的测试中，我们的得到的结果也如预期相符：

！！！！！此处应有vivado里simulation的结果图片！！！！

在 vivado 中简单的进行仿真测试之后，我们将测试代码形成 IP 核并烧入精工板，在精工板上通过数码管进行进一步的仿真测试。

在开发板中，数码管为共阴极数码管，即公共极输入低电平。共阴极由三极管驱动，FPGA需要提供正向信号。同时段选端连接高电平，数码管上的对应位置才可以被点亮。因此，FPGA输出有效的片选信号和段选信号都应该是高电平。
数码管的管脚配置如下：

！！！！这里需要一个图片！！！

将管脚配置完毕之后就可以下板测试了，为了更好的观测结果，我们选择将斐波那契数列结果计算至第二十次进行展示。此时观察精工板测试结果，我们可以明显观察到数码管的数字显示从 1 开始按照斐波那契数列进行递增直至第 20 个结果停止。


# 六. 遇到的问题及解决方法

问题 1：单周期 CPU 代码的实现过程中的模块调用问题，当时最开始想的是在一个模块的结尾调用下一模块，但是这样会很繁琐而且很难控制。
解决方法：设计出一个顶层模块，在顶层模块中对各个底层模块进行调用，易于实现与维护。

问题 2：流水线设计中，R 型指令只需要 IF、ID、EX、WB 四个阶段，不需要经历访存过程，R 型指令与其他类型指令可能出现同时处于 WB 阶段，则寄存器会出现异常
解决方法：通过保证所有指令的执行过程都经历五个阶段，如果一条指令不需要某一阶段的时候，为其加上 NOP 阶段，代表本阶段此指令什么都不干

问题 3：如何判断出现 Data Hazards 的情况及如何解决？
解决方法：通过增加 Forwarding Unit 的数据通路来处理算数运算中的 Data Hazard

问题 4：LW、SW 这一类涉及到存储获取存储器中数据的指令会出现访存冲突
解决方法：引入 NOP 指令，表示这一部分不执行任何指令，这样可以在不大影响流水线 CPU 整体性能的前提下处理访存冲突

问题 5：流水线 CPU 需要知道下一条指令取哪一条，这样流水线才能顺利执行。即控制冲突问题
解决方法：分支预测，在硬件层面预测跳转指令是否会被执行然后按照预测取下一条指令。如果预测失败，那么我们就需要将错误路线上面的指令 Flush 掉，去重新加载正确的指令。

# 七. 设计总结及心得体会

经过了三周的小学期硬件类课程设计，我们从理论课开始，在龙芯公司工程师的指导下回顾了大二大三学习过的计算机组成原理和计算机体系结构的相关知识，并学习了如何将理论知识实践到代码编写中来。两天的理论课程夯实了基础，为接下来的课程设计实验内容做好了铺垫。

课程设计实验包括两个大类，每一类下又分别分为单人项目与小组合作项目。因为小组合作项目实际上是单人项目的深入与递进，因此我们的小组整体实验安排是从单人项目开始，在单人项目中遇到的问题大家合作讨论解决，为之后的小组项目做准备。

小组合作项目中 CPU 的设计可以选择多周期或流水线，我们经过商讨决定选择流水线 CPU 进行设计实现。在实验过程中，我们遇到过许多的问题，但同样的在解决问题的过程中，我们也吸取了经验与教训，汲取了相关知识。虽然我们实现的只是一个简单的五级流水线 CPU，但是却是要实现每一个小部件的功能与要求，只有每一个的部件的功能需求都实现了，最终才可以顺利的进行整个流水，因此在程序的每一个部分都必须深究。

另外纸上得来终觉浅，我们从遇到问题到解决问题，对计算机的组成有了一个大概的理解，同时也掌握了Verilog编程方法，了解了CPU标准。通过自行设计指令集、构造FSM，实现预期效果。通过实验了解了 CPU 是如何理解指令，并加以执行的，以及流水线 CPU 来提升效率的。


# 八. 参考文献
