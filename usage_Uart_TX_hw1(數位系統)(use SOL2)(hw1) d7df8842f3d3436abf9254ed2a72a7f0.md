# usage_Uart_TX_hw1(數位系統)(use SOL2)(hw1)

## usage_Uart_TX_hw1

```verilog
	`include "Uart_TX.v"
	module usage_Uart_TX_hw1(clk_50M, reset_n, write, write_value, uart_txd, finish);
	/*--------input/output--------*/
	input  clk_50M, write, reset_n;
	input  [7:0]write_value;
	output uart_txd, finish;
	/*---------variables---------*/
	reg  [7:0]regdata;
	reg  [2:0]counter;
	reg  [12:0]cnt_uart;
	reg  tick_uart;
	wire TX_D, LDEN, SHEN, rstcount, countEN;
	/*---------assign wire---------*/
	assign uart_txd = (SHEN)?(regdata[0]):(TX_D);
	/*---------9600HZ counter---------*/
	always@(posedge clk_50M or negedge reset_n)begin
		if(!reset_n)cnt_uart <= 13'd0;
		else begin
			if(cnt_uart<13'd5207)cnt_uart <= cnt_uart + 13'd1;
			else cnt_uart <= 13'd0;
		end
	end
	always@(posedge clk_50M or negedge reset_n)begin
		if(!reset_n)tick_uart <= 1'b0;
		else begin
			if(cnt_uart==13'd5206)tick_uart <= 1'b1;
			else                  tick_uart <= 1'b0;
		end
	end
	/*---------Uart_TX instance---------*/
	Uart_TX U0(
		.clk_50M(clk_50M),
		.rst_n(reset_n),
		.count(counter),
		.rstcount(rstcount),
		.countEN(countEN),
		.tick_uart(tick_uart),
		.TX_D(TX_D),
		.LDEN(LDEN),
		.SHEN(SHEN),
		.en(write),
		.finish(finish)
	);
	/*---------counter---------*/
	always@(posedge clk_50M or negedge reset_n)begin
		if(!reset_n)counter <= 3'd0;
		else begin
			if(tick_uart==1'b1)begin
				if(rstcount)    counter <= 3'd0;
				else if(countEN)counter <= counter + 3'd1;
				else            counter <= counter;
			end
			else counter <= counter;
		end
	end
	/*---------write_value---------*/
	always@(posedge clk_50M or negedge reset_n)begin
		if(!reset_n)regdata <= 8'd0;
		else begin
			if(tick_uart==1'b1)begin
				if(LDEN)     regdata <= write_value;
				else if(SHEN)regdata <= regdata>>1;
				else         regdata <= regdata;
			end
			else regdata <= regdata;
		end
	end
	endmodule
```

## Uart_TX module

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
	input tick_uart;
	reg [1:0] fstate;
	assign busy = (fstate!=IDLE);
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

## Wave

![usage_Uart_TX_hw1(%E6%95%B8%E4%BD%8D%E7%B3%BB%E7%B5%B1)(use%20SOL2)%209197c8373eba4578a3ac5ca040a301c6/Untitled.png](usage_Uart_TX_hw1(%E6%95%B8%E4%BD%8D%E7%B3%BB%E7%B5%B1)(use%20SOL2)%209197c8373eba4578a3ac5ca040a301c6/Untitled.png)

## monitor

![usage_Uart_TX_hw1(%E6%95%B8%E4%BD%8D%E7%B3%BB%E7%B5%B1)(use%20SOL2)%209197c8373eba4578a3ac5ca040a301c6/Untitled%201.png](usage_Uart_TX_hw1(%E6%95%B8%E4%BD%8D%E7%B3%BB%E7%B5%B1)(use%20SOL2)%209197c8373eba4578a3ac5ca040a301c6/Untitled%201.png)

## Compilation Report

![usage_Uart_TX_hw1(%E6%95%B8%E4%BD%8D%E7%B3%BB%E7%B5%B1)(use%20SOL2)%209197c8373eba4578a3ac5ca040a301c6/Untitled%202.png](usage_Uart_TX_hw1(%E6%95%B8%E4%BD%8D%E7%B3%BB%E7%B5%B1)(use%20SOL2)%209197c8373eba4578a3ac5ca040a301c6/Untitled%202.png)

## RTL Viewer

![usage_Uart_TX_hw1(%E6%95%B8%E4%BD%8D%E7%B3%BB%E7%B5%B1)(use%20SOL2)%209197c8373eba4578a3ac5ca040a301c6/Untitled%203.png](usage_Uart_TX_hw1(%E6%95%B8%E4%BD%8D%E7%B3%BB%E7%B5%B1)(use%20SOL2)%209197c8373eba4578a3ac5ca040a301c6/Untitled%203.png)

## State Machine

![usage_Uart_TX_hw1(%E6%95%B8%E4%BD%8D%E7%B3%BB%E7%B5%B1)(use%20SOL2)%209197c8373eba4578a3ac5ca040a301c6/Untitled%204.png](usage_Uart_TX_hw1(%E6%95%B8%E4%BD%8D%E7%B3%BB%E7%B5%B1)(use%20SOL2)%209197c8373eba4578a3ac5ca040a301c6/Untitled%204.png)