---
layout: post
title: "systemverilog设计"
subtitle: "Systemverilog设计常用语法"
date: 2024-05-21
author: "Homer"
header-img: "img/post-bg-2015.jpg"
tags: [systemverilog]
---

# 1. 过程块
## 1.1 组合逻辑过程块
always_comb不需要指明敏感列表，使用阻塞赋值(=)，如果过程内部写出锁存器，则会警告或报错

```systemverilog	
always_comb begin
	if(!mode)
		y = a + b;
	else
		y = a - b;
end
```
***注意***：always@(*)并没有组合逻辑的语义，即可能产生latch

***注意***：如果过程块要调用函数，最好用always_comb，因为always@(*)不会推断函数中读取的信号
## 1.2 时序逻辑过程块
always_ff，使用非阻塞赋值(<=)，如果过程内部写出组合逻辑，则会报错
```systemverilog
always_ff @ (posedge clk, negedge rst_n) begin
	if(!rst_n) 
		q <= 0;
	else 
		q <= d;
end
```
## 1.3 锁存逻辑过程块
always_latch，使用非阻塞赋值(<=)，如果写成其他电路，则会警告或报错
```systemverilog
always_latch
	if(en)
		q <= d;
```

# 2. 任务和函数
TODO

# 3. for语句

2种写法：
比如描述一个Reduced-xor电路，如下图

***第1种写法***
```systemverilog
module reduced_xor_gen
	#(parameter N=8)
	(
	input logic [N-1:0] a,
	output logic y
	)
	
	logic [N-1:0] p;

	assign p[0] = a[0];
	
	generate
		genvar i;
		for(i=1; i<N; i=i+1)
			assign p[i] = a[i] ^ p[i-1];
	endgenerate

	assign y = P[N-1];

endmodule
```
第1种写法也是verilog支持的写法

***第2种写法***
```systemverilog
module reduced_xor_gen
	#(parameter N=8)
	(
	input logic [N-1:0] a,
	output logic y
	)
	
	always_comb
	begin
		logic tmp;
		tmp = a[0];
		for (int i=1; i<N; i=i+1)
			tem = a[i] ^ tmp;
		y = tmp;
	end

endmodule
```
第2种写法只在sv中支持，i为声明的局部变量。当for循环启动时，就自动创建这个变量并初始化，当循环推出时，这个变量就被清楚



# 3. 数组，结构体和联合体
## 3.1 结构体
### 结构体定义
```systemverilog
// IEEE 754 Binary 32 encoding
typedef struct packed {
    logic sign;
    logic[FLOAT32_EXP_WIDTH - 1:0] exponent;
    logic[FLOAT32_SIG_WIDTH - 1:0] significand;
} float32_t;
```
### 使用
```systemverilog
float32_t fop1;

assign fop1 = '{1'b1, 8'h12, 23'h111111};

assign s = fop1.sign;
assign exp = fop1.exponent;
assign sign = fop1.significand;
```

TODO

# 4. sv声明的位置
包含以下几个内容：package, $unit编译声明空间, 未命名块中的声明, timescale
## 4.1 package
package中可以包含的可综合的结构有：

1. parameter和localparameter常量定义
2. const常量定义？
3. typedef用户定义类型
4. automatic task和function定义(static类型的task和function不可综合)
5. 从其他包中import语句？
6. 操作数重载定义？

### 4.1.1 定义如下：
```systemverilog
package definitions;

	parameter VERSION = "1.1";
	typedef enum {ADD, SUM, MUL} opcode_t;
	typedef struct {
		logic [31:0] a, b;
		opcode_t opcode;
	} instruction_t;

	function automatic [31:0] mul (input [31:0] a, b);
		return a * b;
	endfunction

endpackage
```
***注意***：包中的参数不能重新定义（verilog中可以对parameter重新定义）
***注意***：包中定义的任务和函数必须声明为automatic，并且不能包含静态变量

### 4.1.2 引用包的内容
4种方式：
1. 用范围解析操作符直接引用
```systemverilog
logic definitions::instruction inst;
```
2. 将包中特定子项导入到模块或接口中
```systemverilog
import definitions::instruction;
logic instruction inst;
```
3. 用通配符导入包中的子项到模块或接口中
```systemverilog
import definitions::*;
logic instruction inst;
always_comb begin
	case(inst.opcode)
		ADD: result = inst.a + inst.b;
		SUB: result = inst.a - inst.b;
		MUL: result = mul(inst.a * inst.b);
	endcase
end
```
4. 将包中子项导入到$unit声明域中
由于不能在关键字module和模块端口定义之间加一个import语句，因此，对于模块端口，包名任须显示引用，例如：

```systemverilog
module ALU
(input definitions::instruction_t inst,
 input logic [31:0] result
)

import definitions::*;
logic instruction inst;
always_comb begin
	case(inst.opcode)
		ADD: result = inst.a + inst.b;
		SUB: result = inst.a - inst.b;
		MUL: result = mul(inst.a * inst.b);
	endcase
end
```
将包中子项导入到$unit声明域中：
```systemverilog
import definitions::*;
module ALU
(input instruction_t inst,
 input logic [31:0] result
)

logic instruction inst;
always_comb begin
	case(inst.opcode)
		ADD: result = inst.a + inst.b;
		SUB: result = inst.a - inst.b;
		MUL: result = mul(inst.a * inst.b);
	endcase
end
```
# 4.2 $unit编译声明空间
***注意***：当多个文件同时进行编译时，只有一个$unit域
当单个文件进行编译时，必须将包中子项导入到$unit域中（每个文件会产生一个$unit域）
当多个文件一起进行编译时，会发生包中子项多次导入的情况（非法），因此，需要使用条件编译
defines.pkg文件如下：
```systemverilog
`ifndef __DEFINES_SVH
`define __DEFINES_SVH

package defines;

endpackage : defines
```
每个需要包中定义的设计都应该将
```systemverilog
`include "defines.pkg"
```
放入文件的开始

# 4.3 未命名块中的声明
verilog允许在命名的begin...end块中声明局部变量
```systemverilog
 module chip (input clock);
 	integer i; // declaration at module level
 	always @(posedge clock)
 		begin: loop
 		// named block
 		integer i;
		// local variable
 		for (i=0; i<=127; i=i+1) begin
 			...
 		end
 	end
 endmodule
```
sv允许在未命名的begin...end块中声明局部变量
```systemverilog
 module chip (input clock);
 	integer i; // declaration at module level
 	always @(posedge clock)
 		begin
 		integer i;
 		// unnamed block 
		// local variable
 		for (i=0; i<=127; i=i+1) begin
 			...
 		end
 	end
 endmodule
```
# 4.4 timescale
TODO
# 5. 层次化设计


# 6. 接口
## 6.1 接口的概念

## 6.2 接口声明
接口可以像模块那样拥有端口

***这样，clock, resetN, test_mode不需要像分立端口那样传递给每个模块实例***

例如：
```systemverilog
interface main_bus (input logic clock, resetN, test_mode);
	wire [15:0] data;
	wire [15:0] addr;
	......
endinterface
```
顶层网表如下：
```systemverilog
module top(input logic clock, resetN, test_mode);
	logic [15:0] pc;
	logic [7:0]  inst;

main_bus bus(
	.clock(clock),
	.resetN(resetN),
	.testmode(test_mode)
);

processor proc1(
	.bus(bus),
	.pc(pc),
	.inst(inst)
);
endmodule
```
## 6.3 将端口用作模块端口

### 6.3.1 显示命名的接口端口
模块端口可以显示的声明为特定接口类型。可以把接口名称当作端口类型使用。
例如：
```systemverilog
interface chip_bus;
......
endinterface

module Cache(chip_bus pins,
			 input clock);
...
endmodule
```
### 6.3.2 通用接口端口
通用接口端用关键字interface定义端口类型，而不是使用特定接口类型的名称。例如：
```systemverilog
module RAM(  interface pins,
			 input clock);
...
endmodule
```
当模块实例化时，任何接口都可以连接到通用接口端口上。

***同一个模块可以用不同的接口连接？***例子？
### 6.3.3 综合指导
接口连接模块的两种方式都是可以综合的。

## 6.4 接口的实例化和连接
***接口类型的端口如果没有连接是非法的***
模块实例的input, output和inout类型的端口可以没有任何连接。对于接口类型的端口则不行。

接口的端口也可以定义为一个接口

## 6.5 接口内部信号的引用
例如
```systemverilog
always @ (posedge bus.clock, negedge bus.resetN)
...
```
## 6.6 接口的modport
***modport定义了，从模块的角度来看接口信号的端口方向***
例如：
```systemverilog
interface chip_bus (input logic clock, resetN);
	logic valid, ready;
	logic [31:0] addr;
	logic [31:0] data;

modport master (input clock,
				input resetN,
				output valid,
				input ready,
				output addr,
				output data);

modport slave (	input clock,
				input resetN,
				input valid,
				output ready,
				input addr,
				input data);
```
modport定义中不包含向量位数和类型。

### 6.6.1 指定使用哪种modport方式
1. 在模块实例的接口连接中说明
```systemverilog
chip_bus bus; //接口实例化
primary i1 (bus.master) //使用master modport
```
2. 在模块定义的端口声明时说明
```systemverilog
module secondary (chip_bus.slave pins)
```
***以上两种方式不能同时使用***

即使接口定义了modport，模块连接到接口时仍然可以不指定modport。
但是，接口内部信号的端口方向只能在modport中定义。
如果没有在接口连接中指定modport，那么接口中所有的线网信号将默认是双向inout信号。

### 6.6.2 使用modport定义不同的连接

模块只可以直接访问在modport定义中列出来的信号。这使得接口中的某些信号可以完全地对某些模块隐藏。

## 6.7 在接口中使用任务和函数
### 6.7.1 接口的方法


### 6.7.2 接口的方法的导入
1. 使用函数和任务的名称名称导入
```systemverilog
modport in(	import Read,
			import parity_gen,
			input clock, resetN);
```
2. 使用函数和任务的完整原型导入
```systemverilog
modport in(	import task Read(input [63:0] data,
							 output [31:0] addr),
			import function parity_gen(input [63:0] data),
			input clock, resetN);
```
***使用从接口中导入的任务或函数的模块是可综合的。***
***导入的函数和任务必须声明为自动类型，并且不能包含静态声明，这样才是可综合的。***

下面是一个完整的例子：
```systemverilog
///
// Simple AHB Interface with parity methods
///
interface simple_ahb (
  input logic  hclk,    // bus transfer clk
  input logic  hresetN  // bus reset, active low
);
  logic [31:0] haddr;   // transfer start address
  logic [32:0] hwdata;  // data to slave, with parity bit
  logic [32:0] hrdata;  // data from slave, with parity bit
  logic [ 2:0] hsize;   // transfer size
  logic        hwrite;  // 1 for write, 0 for read
  logic        hready;  // 1 for transfer finished
 
  function automatic logic parity_gen(logic [31:0] data);
    return(^data); // calculate parity of data (odd parity)
  endfunction
 
  function automatic logic parity_chk(logic [31:0] data,
                                      logic        parity);
    return (parity === ^data); // 1=OK, 0=parity error
  endfunction
 
  // master module port directions
  modport master_ports (
    output haddr, hwdata, hsize, hwrite, // to AHB slave
    input  hrdata, hready,               // from AHB slave
    input  hclk, hresetN,                // from chip level
    import parity_gen, parity_check      // function import
  );
 
  // slave module port directions
  modport slave_ports (
    output hrdata, hready,               // to AHB master
    input  haddr, hwdata, hsize, hwrite, // from AHB master
    input  hclk, hresetN,                // from chip level
    import parity_check                  // function import
  );
 
  // slave module port directions
  modport slave_ports_alt (
    output hrdata, hready,               // to AHB master
    input  haddr, hwdata, hsize, hwrite, // from AHB master
    input  hclk, hresetN,                // from chip level
    import function logic parity_chk(logic [31:0] data,
                                     logic        parity)
  );
endinterface: simple_ahb
```

## 6.8 可重构接口
### 6.8.1 参数化接口

```systemverilog
///
// Simple AHB Interface with pareterized bus widths
///
interface simple_ahb 
#(parameter DWIDTH=32)        // Data bus width
(
  input logic  hclk,          // bus transfer clk
  input logic  hresetN        // bus reset, active low
);
  logic [31:0]       haddr;   // transfer start address
  logic [DWIDTH-1:0] hwdata;  // data sent to slave
  logic [DWIDTH-1:0] hrdata;  // return data from slave
  logic [ 2:0]       hsize;   // transfer size             
  logic              hwrite;  // 1 for write, 0 for read
  logic              hready;  // 1 for transfer finished
 
  // master module port directions
  modport master_ports (
    output haddr, hwdata, hsize, hwrite, // to AHB slave
    input  hrdata, hready,               // from AHB slave
    input  hclk, hresetN                 // from chip level
  );
 
  // slave module port directions
  modport slave_ports (
    output hrdata, hready,               // to AHB master
    input  haddr, hwdata, hsize, hwrite, // from AHB master
    input  hclk, hresetN                 // from chip level
  );
endinterface: simple_ahb
```
例化
```systemverilog
simple_ahb #(.DWIDTH(64)) 
	ahbl(.hclk,
		 .hresetN
) ;
```

### 6.8.2 使用generate块
TODO

