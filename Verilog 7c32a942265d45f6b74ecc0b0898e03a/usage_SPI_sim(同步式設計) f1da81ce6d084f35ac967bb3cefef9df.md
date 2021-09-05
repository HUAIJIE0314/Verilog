# usage_SPI_sim(同步式設計)

```verilog
`include "SPI.v"
	module usage_SPI_sim(clk_sys, rst_n, regdata, CS, SCLK, SDA, ready, finish);
	/*-------in/output------*/
	input       [15:0]regdata;
	input       clk_sys, rst_n, ready;
	output wire SDA, CS, SCLK, finish;
	/*-------variables------*/
	reg [3:0]counter = 4'd0;
	wire countEN, rstcount, SHEN, LDEN, SCLK_temp, tick_SPI;
	reg [15:0]Q;
	/*-------assign wire------*/
	assign SDA = (SHEN)?(Q[15]):(1'b0);
	assign SCLK_temp = clk_sys;
	assign tick_SPI = 1'b1;
	/*-------ctrl counter------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)begin
			counter <= 4'd0;
		end
		else begin
			if(rstcount)    counter <= 4'd0;
			else if(countEN)counter <= counter + 4'd1;
			else            counter <= 4'd0;
		end
	end
	/*-------main------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)begin
			Q <= 16'd0;
		end
		else begin
			if(LDEN)     Q <= regdata;
			else if(SHEN)Q[15:0] <= {Q[14:0], 1'b0};
			else         Q <= 16'd0;
		end
	end
	SPI U0(
		.clk_sys(clk_sys),
		.SCLK_temp(SCLK_temp),
		.tick_SPI(tick_SPI),
		.rst_n(rst_n),
		.SCLK(SCLK),
		.CS(CS),
		.count(counter),
		.countEN(countEN),
		.rstcount(rstcount),
		.ready(ready),
		.finish(finish),
		.SHEN(SHEN),
		.LDEN(LDEN)
	);
	endmodule
```

## SPI

```verilog
	module SPI(clk_sys, SCLK_temp, tick_SPI, rst_n, SCLK, CS, count, countEN, rstcount, ready, finish, SHEN, LDEN);
	/*-----------parameter-----------*/
	parameter IDLE  = 2'd0;
	parameter START = 2'd1;
	parameter SHITF = 2'd2;
	parameter STOP  = 2'd3;
	/*-----------in/output-----------*/
	input clk_sys, SCLK_temp, rst_n, ready, tick_SPI;
	input [3:0]count;
	output reg	CS, countEN, rstcount, SHEN, LDEN, finish;
	output wire SCLK;
	/*-----------variables-----------*/
	reg [1:0]fstate = 2'd0;
	/*-----------assign-----------*/
	assign SCLK = (SHEN)?(~SCLK_temp):(1'b1);
	/*-----------state-----------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)fstate <= IDLE;
		else begin
			case(fstate)
				IDLE:begin
					if(ready)fstate <= START;
					else     fstate <= IDLE;
				end
				START:begin
					if(tick_SPI)fstate <= SHITF;
					else        fstate <= START;
				end
				SHITF:begin
					if(count==4'd14&&tick_SPI)fstate <= STOP;
					else                      fstate <= SHITF;
				end
				STOP:begin
					if(tick_SPI)fstate <= IDLE;
					else        fstate <= STOP;
				end
				default:fstate <= IDLE;
			endcase
		end
	end
	/*-----------output-----------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)begin
			CS       <= 1'b1;
			countEN  <= 1'b0;
			rstcount <= 1'b0;
			SHEN     <= 1'b0;
			LDEN     <= 1'b0;
			//finish   <= 1'b0;
		end
		else begin
			if(tick_SPI)begin
				case(fstate)
					IDLE:begin
						CS       <= 1'b1;
						countEN  <= 1'b0;
						rstcount <= 1'b0;
						SHEN     <= 1'b0;
						LDEN     <= 1'b0;
						//finish   <= 1'b0;
					end
					START:begin
						CS       <= 1'b0;
						countEN  <= 1'b0;
						rstcount <= 1'b0;
						SHEN     <= 1'b0;
						LDEN     <= 1'b1;
						//finish   <= 1'b0;
					end
					SHITF:begin
						CS       <= 1'b0;
						countEN  <= 1'b1;
						if(count==4'd14)    rstcount  <= 1'b1;
						else if(count<4'd14)rstcount  <= 1'b0;
						else                rstcount  <= 1'b0;//prevent latch
						SHEN     <= 1'b1;
						LDEN     <= 1'b0;
						//finish   <= 1'b0;
					end
					STOP:begin
						CS       <= 1'b1;
						countEN  <= 1'b0;
						rstcount <= 1'b0;
						SHEN     <= 1'b0;
						LDEN     <= 1'b0;
						//finish   <= 1'b1;
					end
					default:begin
						CS       <= 1'bx;
						countEN  <= 1'bx;
						rstcount <= 1'bx;
						SHEN     <= 1'bx;
						LDEN     <= 1'bx;
						//finish   <= 1'bx;
					end
				endcase
			end
			else begin
				CS       <= CS;
				countEN  <= countEN;
				rstcount <= rstcount;
				SHEN     <= SHEN;
				LDEN     <= LDEN;
				//finish   <= finish;
			end
		end
	end
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)begin
			finish <= 1'b0;
		end
		else begin
			if(fstate==STOP&&tick_SPI)finish <= 1'b1;
			else                      finish <= 1'b0;
		end
	end
	endmodule
```

## TestBench

```verilog
`timescale 10ns/1ns
	module tb_usage_SPI_sim();
	
	reg       [15:0]regdata;
	reg       clk_sys, rst_n, ready;
	wire      SDA, CS, SCLK, finish;
	
	usage_SPI_sim UUT(
		.clk_sys(clk_sys),
		.rst_n(rst_n),
		.regdata(regdata),
		.CS(CS),
		.SCLK(SCLK),
		.SDA(SDA),
		.ready(ready),
		.finish(finish)
	);
	reg flag = 1'b0;
	initial begin
		clk_sys = 1'b0; 
		regdata = 16'b1010100110101010;
		rst_n = 1'b0;
		#100;
		rst_n = 1'b1;
	end
	initial begin
		ready = 1'b0;
		flag = 1'b1;
		#100 ready = 1'b1;
		#100 ready = 1'b0;
		flag = 1'b1;
	end
	always begin
		#50 clk_sys = ~clk_sys;
	end
	
	always@(posedge clk_sys)begin
		if(flag)begin
			if(finish)begin
				regdata[15:0] <= {regdata[14:0], regdata[15]};
				#100 ready <= 1'b1;
				#100 ready <= 1'b0;
			end
		end
	end
	
	endmodule
```

## Wave

![usage_SPI_sim(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)%20f1da81ce6d084f35ac967bb3cefef9df/Untitled.png](usage_SPI_sim(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)%20f1da81ce6d084f35ac967bb3cefef9df/Untitled.png)

![usage_SPI_sim(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)%20f1da81ce6d084f35ac967bb3cefef9df/Untitled%201.png](usage_SPI_sim(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)%20f1da81ce6d084f35ac967bb3cefef9df/Untitled%201.png)

## Compilation Report

![usage_SPI_sim(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)%20f1da81ce6d084f35ac967bb3cefef9df/Untitled%202.png](usage_SPI_sim(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)%20f1da81ce6d084f35ac967bb3cefef9df/Untitled%202.png)

## RTL Viewer

![usage_SPI_sim(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)%20f1da81ce6d084f35ac967bb3cefef9df/Untitled%203.png](usage_SPI_sim(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)%20f1da81ce6d084f35ac967bb3cefef9df/Untitled%203.png)

## State Machine

![usage_SPI_sim(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)%20f1da81ce6d084f35ac967bb3cefef9df/Untitled%204.png](usage_SPI_sim(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)%20f1da81ce6d084f35ac967bb3cefef9df/Untitled%204.png)