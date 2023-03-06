默认情况下，chisel3 生成的 verilog 代码在 Vivado 中仿真会出现很多信号大面积变成 X。
```verilog
`define RANDOMIZE_REG_INIT
`define RANDOMIZE_MEM_INIT
`define RANDOMIZE_GARBAGE_ASSIGN
`define RANDOMIZE_INVALID_ASSIGN
```
在生成的 verilog 前面加上这四句，就可以正常仿真了。

使用寄存器可以对按键的输入进行时钟同步
