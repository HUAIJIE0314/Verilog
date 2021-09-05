# SPI(同步式設計+finish持續一個clk_sys)

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
	assign SCLK = (fstate==SHITF)?(~SCLK_temp):(1'b1);
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
					if(count==4'd15&&tick_SPI)fstate <= STOP;
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
						if(count==4'd15)    rstcount  <= 1'b1;
						else if(count<4'd15)rstcount  <= 1'b0;
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

---

## TestBench

```verilog
	`timescale 10ns/1ns
	module tb_SPI();
	
	reg clk_sys, SCLK_temp, tick_SPI, rst_n, ready;
	reg [3:0]count;
	wire CS, countEN, rstcount, SHEN, LDEN, finish;
	wire SCLK;
	reg [5:0]cnt_SPI;
	
	SPI U0(
		.clk_sys(clk_sys),
		.SCLK_temp(SCLK_temp), 
		.tick_SPI(tick_SPI),
		.rst_n(rst_n), 
		.SCLK(SCLK), 
		.CS(CS), 
		.count(count), 
		.countEN(countEN), 
		.rstcount(rstcount), 
		.ready(ready), 
		.finish(finish), 
		.SHEN(SHEN), 
		.LDEN(LDEN)
	);
	initial begin
		clk_sys = 0;
		SCLK_temp = 0;
   end
	initial begin
		while(1)
			#5 clk_sys = ~clk_sys;
	end
	initial begin
       rst_n = 0;
       while(1)
           #100 rst_n = 1;
   end
	initial begin
       ready = 0;
       #100 ready = 1;
       #100 ready = 0;
   end
	/*-------1M counter------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)cnt_SPI <= 6'd0;
		else begin
			if(cnt_SPI<6'd49)begin//0-49
				cnt_SPI  <= cnt_SPI + 6'd1;
			end
			else begin
				cnt_SPI  <= 6'd0;
			end
		end
	end
	/*-------1M tick------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)tick_SPI <= 1'b0;
		else begin
			if(cnt_SPI==6'd48)tick_SPI <= 1'b1;
			else              tick_SPI <= 1'b0;
		end
	end
	/*-------ctrl counter------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)begin
			count <= 4'd0;
		end
		else begin
			if(tick_SPI)begin
				if(rstcount)    count <= 4'd0;
				else if(countEN)count <= count + 4'd1;
				else            count <= 4'd0;
			end
			else count <= count;
		end
	end
	/*-------make SCLK------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)begin
			SCLK_temp <= 1'b0;
		end
		else begin
			if(cnt_SPI == 6'd24)     SCLK_temp <= 1'b0;
			else if(cnt_SPI == 6'd49)SCLK_temp <= 1'b1;
			else                     SCLK_temp <= SCLK_temp;
		end
	end
	endmodule
```

## Wave

![SPI(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88+finish%E6%8C%81%E7%BA%8C%E4%B8%80%E5%80%8Bclk_sys)%2082387958fce242c9bc6a1adc3e84fea3/Untitled.png](SPI(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88+finish%E6%8C%81%E7%BA%8C%E4%B8%80%E5%80%8Bclk_sys)%2082387958fce242c9bc6a1adc3e84fea3/Untitled.png)

## Compilation Report

![SPI(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88+finish%E6%8C%81%E7%BA%8C%E4%B8%80%E5%80%8Bclk_sys)%2082387958fce242c9bc6a1adc3e84fea3/Untitled%201.png](SPI(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88+finish%E6%8C%81%E7%BA%8C%E4%B8%80%E5%80%8Bclk_sys)%2082387958fce242c9bc6a1adc3e84fea3/Untitled%201.png)

## RTL Viewer

![SPI(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88+finish%E6%8C%81%E7%BA%8C%E4%B8%80%E5%80%8Bclk_sys)%2082387958fce242c9bc6a1adc3e84fea3/Untitled%202.png](SPI(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88+finish%E6%8C%81%E7%BA%8C%E4%B8%80%E5%80%8Bclk_sys)%2082387958fce242c9bc6a1adc3e84fea3/Untitled%202.png)

## State Machine

![SPI(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88+finish%E6%8C%81%E7%BA%8C%E4%B8%80%E5%80%8Bclk_sys)%2082387958fce242c9bc6a1adc3e84fea3/Untitled%203.png](SPI(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88+finish%E6%8C%81%E7%BA%8C%E4%B8%80%E5%80%8Bclk_sys)%2082387958fce242c9bc6a1adc3e84fea3/Untitled%203.png)