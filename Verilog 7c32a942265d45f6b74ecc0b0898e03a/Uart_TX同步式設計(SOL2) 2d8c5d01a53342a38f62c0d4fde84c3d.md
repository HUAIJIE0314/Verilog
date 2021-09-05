# Uart_TX同步式設計(SOL2)

## Uart_TX

```verilog
module Uart_TX(tick_uart, clk_50M, rst_n, count, rstcount, countEN, TX_D, LDEN, SHEN, en, busy, finish);
	/*---------parameter---------*/
	parameter IDLE  = 2'd0;
	parameter START = 2'd1;
	parameter SHIFT = 2'd2;
	parameter STOP  = 2'd3;
	/*---------ports declaration---------*/
	input       clk_50M, rst_n, en;
	input     	[2:0]count;
	output reg  TX_D, LDEN, SHEN, rstcount, countEN, finish;
	output wire busy;
	/*---------variables---------*/
	//reg [12:0]cnt_9600 = 13'd0;
	input tick_uart;
	reg [1:0] fstate;
	assign busy = (fstate!=IDLE);
	/*---------9600HZ counter---------*/
	/*
	always@(posedge clk_50M or negedge rst_n)begin
		if(!rst_n)cnt_9600 <= 13'd0;
		else begin
			if(cnt_9600 < 13'd5208)cnt_9600 <= cnt_9600 + 13'd1;
			else cnt_9600 <= 13'd0;
		end
	end
	*/
	/*---------fstate state---------*/
	always@(posedge clk_50M or negedge rst_n)begin
		if(!rst_n)fstate <= IDLE;
		else begin
			case(fstate)
				IDLE:begin
					if(en)fstate <= START;
					else  fstate <= IDLE;
				end
				START:begin
					if(tick_uart==1'b1)fstate <= SHIFT;
					else               fstate <= START;
				end
				SHIFT:begin
					if(tick_uart==1'b1&&count==3'd6)fstate <= STOP;
					else                            fstate <= SHIFT;
				end
				STOP:begin
					if(tick_uart==1'b1)fstate <= IDLE;
					else               fstate <= STOP;
				end
				default:fstate <= IDLE;
			endcase
		end
	end
	/*---------fstate output---------*/
	always@(posedge clk_50M or negedge rst_n)begin
		if(!rst_n)begin
			TX_D     <= 1'b1;
			SHEN     <= 1'b0;
			LDEN     <= 1'b0;
			rstcount <= 1'b0;
			countEN  <= 1'b0;
		end 
		else begin
			if(tick_uart==1'b1)begin
				case(fstate)
					IDLE:begin
						TX_D     <= 1'b1;
						SHEN     <= 1'b0;
						LDEN     <= 1'b0;
						rstcount <= 1'b0;
						countEN  <= 1'b0;
					end
					START:begin
						TX_D     <= 1'b0;
						SHEN     <= 1'b0;
						LDEN     <= 1'b1;
						rstcount <= 1'b0;
						countEN  <= 1'b0;
					end
					SHIFT:begin
						TX_D     <= 1'bz;
						SHEN     <= 1'b1;
						LDEN     <= 1'b0;
						if(count==3'd6)    rstcount <= 1'b1;
						else if(count<3'd6)rstcount <= 1'b0;
						else               rstcount <= 1'b0;//prevent latch
						countEN  <= 1'b1;
					end
					STOP:begin
						TX_D     <= 1'b1;
						SHEN     <= 1'b0;
						LDEN     <= 1'b0;
						rstcount <= 1'b0;
						countEN  <= 1'b0;
					end
					default:begin
						TX_D     <= 1'b1;
						SHEN     <= 1'b0;
						LDEN     <= 1'b0;
						rstcount <= 1'b0;
						countEN  <= 1'b0;
					end
				endcase
			end
			else begin
				TX_D     <= TX_D;
				SHEN     <= SHEN;
				LDEN     <= LDEN;
				rstcount <= rstcount;
				countEN  <= countEN;
			end
		end
	end
	always@(posedge clk_50M or negedge rst_n)begin
		if(!rst_n)finish <= 1'b0;
		else begin
			if(fstate==STOP&&tick_uart)finish <= 1'b1;
			else                       finish <= 1'b0;
		end
	end
	endmodule
```

## TestBench

```verilog
`timescale 10ns/1ns
	module Uart_TX_tb();
	reg  tick_uart;
	reg  clk_50M, rst_n, en;
	reg  [2:0]count;
	wire TX_D, LDEN, SHEN, rstcount, countEN;
	wire busy, finish;
	Uart_TX UUT(
		.tick_uart(tick_uart),
		.clk_50M(clk_50M),
		.rst_n(rst_n),
		.count(count),
		.rstcount(rstcount),
		.countEN(countEN),
		.TX_D(TX_D),
		.LDEN(LDEN),
		.SHEN(SHEN),
		.en(en),
		.busy(busy),
		.finish(finish)
	);
	always begin
		#5 clk_50M = ~clk_50M;
	end
	initial begin
		rst_n = 1'b0;
		clk_50M = 1'b0;
		tick_uart = 1'b1;
		#100 rst_n = 1'b1;
	end
	initial begin
		en = 1'b0;
		#100 en = 1'b1;
		#100 en = 1'b0;
	end
	initial begin
		count = 0;
		#200 count = 1;
		#100 count = 2;
		#100 count = 3;
		#100 count = 4;
		#100 count = 5;
		#100 count = 6;
		#100 count = 7;
		#2000;
		$finish;
	end
	
	endmodule
```

## Wave

![Uart_TX%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88(SOL2)%202d8c5d01a53342a38f62c0d4fde84c3d/Untitled.png](Uart_TX%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88(SOL2)%202d8c5d01a53342a38f62c0d4fde84c3d/Untitled.png)

## Compilation Report

![Uart_TX%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88(SOL2)%202d8c5d01a53342a38f62c0d4fde84c3d/Untitled%201.png](Uart_TX%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88(SOL2)%202d8c5d01a53342a38f62c0d4fde84c3d/Untitled%201.png)

## RTL Viewer

![Uart_TX%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88(SOL2)%202d8c5d01a53342a38f62c0d4fde84c3d/Untitled%202.png](Uart_TX%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88(SOL2)%202d8c5d01a53342a38f62c0d4fde84c3d/Untitled%202.png)

## State machine

![Uart_TX%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88(SOL2)%202d8c5d01a53342a38f62c0d4fde84c3d/Untitled%203.png](Uart_TX%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88(SOL2)%202d8c5d01a53342a38f62c0d4fde84c3d/Untitled%203.png)

## Note

SHEN：Shift enable

LDEN：Load enable