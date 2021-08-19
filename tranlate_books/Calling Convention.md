# Calling Convention 

本章描述了RV32和RV64程序的C编译器标准和两个调用约定：附加标准通用扩展（RV32G/RV64G）的基础ISA约定，以及缺乏浮点单元（例如RV32I/RV64I）实现的软浮点约定。

> 使用ISA扩展的实现可能需要扩展调用约定。

# 18.1 C语言的数据类型和对齐方式

表18.1总结了RISC-V C程序本机支持的数据类型。在RV32和RV64 C编译器中，C中的`int`类型都是32位。另一方面，`long`和指针都与整数寄存器位数一致，所以在RV32中，两者都是32位，而在RV64中，两者都是64位。同样，RV32采用ILP32整数模型，而RV64是LP64。在RV32和RV64中，C类型`long long`是64位整数，`float`是遵循IEEE754-2008标准的32位浮点数，`double`是遵循IEEE754-2008标准的64位浮点数，`long double`是遵循IEEE754-2008标准的128位浮点数。

C类型`char`和`unsigned char`都是8位无符号整数，当存储在RISC-V整数寄存器中时是零扩展。`unsigned short`是16位无符号整数，当存储在RISC-V整数寄存器中时是零扩展。`signed char`是8位有符号整数，当存储在RISC-V整数寄存器中时是符号扩展的，即比特位从（XLEN-1）到7都是相等的。`short`是16位有符号整数，当存储在寄存器中时是符号扩展的。

在RV64中，32位的数据类型（如`int`）以合适的符号扩展存储在整数寄存器中；也就是说，比特位从63到31都是相等的。即使是无符号的32位类型，这个限制也适用。

RV32和RV64 C编译器和兼容软件将所有上述数据类型存储在内存中时保持自然对齐。

| **C数据类型** | **描述**       | **RV32中字节数** | **RV64中字节数** |
| ------------- | -------------- | ---------------- | ---------------- |
| `char`        | 字符值/字节    | 1                | 1                |
| `short`       | 短整型         | 2                | 2                |
| `int`         | 整型           | 4                | 4                |
| `long`        | 长整型         | 4                | 8                |
| `long long`   | 超长整型       | 8                | 8                |
| `void*`       | 指针           | 4                | 8                |
| `float`       | 单精度浮点型   | 4                | 4                |
| `double`      | 双精度浮点型   | 8                | 8                |
| `long double` | 扩展精度浮点型 | 16               | 16               |

表18.1：基于RISC-V指令集的C编译器数据类型

# 18.2 RVG调用协定

RISC-V调用约定尽可能在寄存器中传递参数。为此，最多使用八个整数寄存器**a0**-**a7**和八个浮点寄存器**fa0**-**fa7**。

如果函数的参数被概念化为C结构体的字段，结构体中的每个字段都按指针长度对齐，则参数寄存器是该结构体中前八个指针字长参数的副本。如果第i(i<8)个参数是浮点类型，则在浮点寄存器**fai**中传递；否则，在整数寄存器**ai**中传递。但是，浮点参数如果属于`union`或结构体中数组字段的一部分，就会在整数寄存器中传递。此外，变参函数的浮点参数（未显式命名参数列表的函数）在整数寄存器中传递。

小于指针字长的参数在参数寄存器的最低有效位(LSB)中传递。相应地，栈上传递的小于指针字长的参数出现在指针字的较低地址中，因为RISC-V有一个小端存储系统。

当在堆栈上传递两倍于指针字大小的基本参数时，它们是自然对齐的。当它们在整数寄存器中传递时，它们驻留在对齐的偶数号-奇数号寄存器对中，偶数寄存器保存最低有效位。例如，在RV32中，函数`void foo(int, long long)`的第一个参数在**a0**中传递，第二个参数在**a2**和**a3**中传递。**a1**中不传递任何内容。

大于指针字大小两倍的参数通过引用传递。

结构体中未在参数寄存器中传递的部分在栈上传递。栈指针**sp**指向未在寄存器中传递的第一个参数。

函数在整数寄存器**a0**和**a1**以及浮点寄存器**fa0**和**fa1**中返回值。只有当浮点值是原始值（传入时**fa0**和**fa1**作为参数寄存器，原始值是指该参数不改变而直接返回）或作为仅有一两个浮点值组成的结构体的成员时，才会从浮点寄存器中返回。长度恰好为两个指针字长的其他返回值将在**a0**和**a1**中返回。较大的返回值完全在内存中传递；调用方分配此内存区域，并将指针作为隐式的第一个参数传递给被调用方。

在标准的RISC-V调用约定中，栈向下增长，栈指针始终保持16字节对齐。

除了自变量和返回值寄存器之外，还有在调用中不稳定的七个整数寄存器**t0**-**t6**和十二个浮点寄存器**ft0**-**ft11**作为临时寄存器，如果之后使用，调用者必须保存它们。十二个整数寄存器**s0**-**s11**和十二个浮点寄存器**fs0**-**fs11**在调用中受保护，如果使用，被调用者必须保存它们。表18.2显示了调用约定中每个整数和浮点寄存器的作用。

| **寄存器** | **ABI名称** | **描述**          | **保存者** |
| ---------- | ----------- | ----------------- | ---------- |
| x0         | zero        | 硬布线零          |            |
| x1         | ra          | 返回地址          | 调用者     |
| x2         | sp          | 栈指针            | 被调用者   |
| x3         | gp          | 全局指针          |            |
| x4         | tp          | 线程指针          |            |
| x5-7       | t0-2        | 临时暂存单元      | 调用者     |
| x8         | s0/fp       | 保留寄存器/帧指针 | 被调用者   |
| x9         | s1          | 保留寄存器        | 被调用者   |
| x10-11     | a0-1        | 函数参数/返回值   | 调用者     |
| x12-17     | a2-7        | 函数参数          | 调用者     |
| x18-27     | s2-11       | 保留寄存器        | 被调用者   |
| x28-31     | t3-6        | 临时暂存单元      | 调用者·    |
| f0-7       | ft0-7       | 浮点临时暂存单元  | 调用者     |
| f8-9       | fs0-1       | 浮点保留寄存器    | 被调用者   |
| f10-11     | fa0-1       | 浮点参数/返回值   | 调用者     |
| f12-17     | fa2-7       | 浮点参数          | 调用者     |
| f18-27     | fs2-11      | 浮点保留寄存器    | 被调用者   |
| f28-31     | ft8-11      | 浮点临时暂存单元  | 调用者     |

表18.2 RISC-V调用协定寄存器的使用

# 18.3 软浮点数调用协定

软浮点调用约定用于缺乏浮点硬件的RV32和RV64实现。它避免使用了F、D和Q标准扩展中的所有指令，从而避免使用**f**寄存器。

完整参数的传递和返回方式与RVG约定相同，栈规则也相同。浮点参数使用长度相同的整型参数的规则在整数寄存器中传递和返回。例如，在RV32中，函数`double foo(int, double, long double)`的第一个参数在**a0**中传递，第二个参数在**a2**和**a3**中传递，第三个参数通过**a4**传递引用；其结果在**a0**和**a1**中返回。在RV64中，参数以**a0**、**a1**和**a2**-**a3**对形式传递，结果以**a0**形式返回。

动态舍入模式和累计异常标志可以通过C99头文件***fenv.h***提供的程序访问。

> 注：为了编写高精度浮点数的运算，编程人员需要控制浮点数环境的各个方面：结果如何舍入，浮点数表达式如何简化与变换，如何处理浮点数异常（如下溢之类的浮点数异常是忽略还是产生错误）等等。C99引入了***fenv.h***来控制浮点数环境。