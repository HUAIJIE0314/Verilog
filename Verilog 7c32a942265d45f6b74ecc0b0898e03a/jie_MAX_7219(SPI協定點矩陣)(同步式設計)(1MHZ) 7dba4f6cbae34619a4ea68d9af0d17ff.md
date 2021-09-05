# jie_MAX_7219(SPI協定點矩陣)(同步式設計)(1MHZ)

## jie_MAX_7219

```verilog
`include "edge_detect.v"
	`include "SPI.v"
	module jie_MAX_7219(clk_sys, rst_n, CS, SCLK, SDA);
	/*-------in/output------*/
	input       clk_sys, rst_n;
	output wire SDA, CS, SCLK;
	/*-------variables------*/
	reg [3:0]counter = 4'd0;
	wire ready;
	wire countEN, rstcount, SHEN, LDEN, finish;
	reg [15:0]Q;
	reg [15:0]regdata;
	reg [3:0]index = 4'd0;
	reg [5:0]cnt_SPI = 6'd0;
	reg tick_SPI  = 1'b0;
	reg SCLK_temp = 1'b0;
	wire pos_edge, neg_edge;
	/*-------assign wire------*/
	assign SDA = (SHEN)?(Q[15]):(1'b0);
	assign ready = 1'b1;
	/*-------module instantiate------*/
	SPI U0(
		.clk_sys(clk_sys),
		.SCLK_temp(SCLK_temp), 
		.rst_n(rst_n),
		.tick_SPI(tick_SPI),
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
	edge_detect U1(
		.clk(clk_sys),
		.rst_n(rst_n),
		.data_in(finish),
		.pos_edge(pos_edge),
		.neg_edge(neg_edge) 
   );
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
	/*-------ctrl counter------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)begin
			counter <= 4'd0;
		end
		else begin
			if(tick_SPI)begin
				if(rstcount)    counter <= 4'd0;
				else if(countEN)counter <= counter + 4'd1;
				else            counter <= 4'd0;
			end
			else counter <= counter;
		end
	end
	/*-------send data------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)begin
			Q <= 16'd0;
		end
		else begin
			if(tick_SPI)begin
				if(LDEN)     Q <= regdata;
				else if(SHEN)Q[15:0] <= {Q[14:0], 1'b0};
				else         Q <= 16'd0;
			end
			else Q <= Q;
		end
	end
	/*-------instruction/data------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)begin
			regdata <= {8'h00, 8'h00};
		end
		else begin
			if(tick_SPI)begin
				case(index)
					4'h0:regdata <= {8'h00, 8'h00};
					4'h1:regdata <= {8'h01, 8'b00000000};
					4'h2:regdata <= {8'h02, 8'b01100110};
					4'h3:regdata <= {8'h03, 8'b11111111};
					4'h4:regdata <= {8'h04, 8'b11111111};
					4'h5:regdata <= {8'h05, 8'b11111111};
					4'h6:regdata <= {8'h06, 8'b01111110};
					4'h7:regdata <= {8'h07, 8'b00111100};
					4'h8:regdata <= {8'h08, 8'b00011000};
					4'h9:regdata <= {8'h09, 8'h00};//decode
					4'hA:regdata <= {8'h0A, 8'h00};//light
					4'hB:regdata <= {8'h0B, 8'h07};//scanline
					4'hC:regdata <= {8'h0C, 8'h01};//shutdown
					//4'hD:regdata <= {8'h0D, 8'h00};
					//4'hE:regdata <= {8'h0E, 8'h00};
					4'hF:regdata <= {8'h0F, 8'h00};
					default:regdata <= {8'h00, 8'h00};
				endcase
			end
			else regdata <= regdata;
		end
	end
	/*-------instruction/data's index------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)index <= 4'd0;
		else begin
			if(pos_edge)index <= index + 4'd1;
			else        index <= index;
		end 
	end
	endmodule
```

## SPI Module

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

## edge_detect

```verilog
module edge_detect (
   clk,
   rst_n,
   data_in,
   pos_edge,
   neg_edge 
   );
input      clk;
input      rst_n;
input      data_in;
output     pos_edge;
output     neg_edge;

reg        data_in_d1;
reg        data_in_d2; 

assign pos_edge =  data_in_d1 & ~data_in_d2;
assign neg_edge =  ~data_in_d1 & data_in_d2;

 
always@(posedge clk or negedge rst_n) 
begin
  if (!rst_n)
  begin
    data_in_d1 <= 1'b0;
    data_in_d2 <= 1'b0;
  end
  else 
  begin
    data_in_d1 <= data_in;
    data_in_d2 <= data_in_d1;   
  end
end

endmodule
```

## TestBench

```verilog
`timescale 10ns/1ns
	module tb_jie_MAX_7219();
	reg  clk_sys, rst;
	wire SDA, CS, SCLK;
	jie_MAX_7219 UUT(
		.clk_sys(clk_sys),
		.rst_n(rst),
		.CS(CS),
		.SCLK(SCLK),
		.SDA(SDA)
	);
	initial begin
		clk_sys <= 1'b0;
		rst <= 1'b0;
		repeat(5)@(posedge clk_sys)rst <= 1'b0;
		rst <= 1'b1;
	end
	always begin
		#5 clk_sys <= ~clk_sys;
	end
	endmodule
```

## Wave

![jie_MAX_7219(SPI%E5%8D%94%E5%AE%9A%E9%BB%9E%E7%9F%A9%E9%99%A3)(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(1MHZ)%207dba4f6cbae34619a4ea68d9af0d17ff/Untitled.png](jie_MAX_7219(SPI%E5%8D%94%E5%AE%9A%E9%BB%9E%E7%9F%A9%E9%99%A3)(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(1MHZ)%207dba4f6cbae34619a4ea68d9af0d17ff/Untitled.png)

## Compilation Report

![jie_MAX_7219(SPI%E5%8D%94%E5%AE%9A%E9%BB%9E%E7%9F%A9%E9%99%A3)(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(1MHZ)%207dba4f6cbae34619a4ea68d9af0d17ff/Untitled%201.png](jie_MAX_7219(SPI%E5%8D%94%E5%AE%9A%E9%BB%9E%E7%9F%A9%E9%99%A3)(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(1MHZ)%207dba4f6cbae34619a4ea68d9af0d17ff/Untitled%201.png)