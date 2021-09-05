# I2C_control(18bit)(sol2)(未含tick)

## I2C_control

```verilog
	module I2C_control_18bit(clk_sys, rst_n, en, count, countEN, rstcount, ACK1, ACK2, rstACK, SCLK, SCLK_temp,  SHEN, LDEN, SDO);
	/*---------parameter---------*/
	parameter IDLE  = 3'd0;
	parameter GO    = 3'd1;
	parameter START = 3'd2;
	parameter WAIT  = 3'd3;
	parameter SHIFT = 3'd4;
	parameter STOP  = 3'd5;
	parameter FINAL = 3'd6;
	parameter END   = 3'd7;
	/*---------ports declaration---------*/
	input       clk_sys, rst_n, en;
	input       [4:0]count;
	output reg  ACK1, ACK2, SCLK_temp, SHEN, LDEN, SDO, countEN, rstcount, rstACK;
	output wire SCLK;
	/*---------variables---------*/
	reg [2:0]fstate;
	/*---------assign wire---------*/
	assign SCLK = (SHEN)?(~clk_sys):(SCLK_temp);
	/*---------fstate state---------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)begin
			fstate <= IDLE;
		end
		else begin
			case(fstate)
				IDLE:begin
					if(en) fstate <= GO;
					else   fstate <= IDLE;
				end
				GO:begin
					fstate <= START;
				end
				START:begin
					fstate <= WAIT;
				end
				WAIT:begin
					fstate <= SHIFT;
				end
				SHIFT:begin
					if(count==5'd17)fstate <= STOP;
					else            fstate <= SHIFT;
				end
				STOP:begin
					fstate <= FINAL;
				end
				FINAL:begin
					fstate <= END;
				end
				END:begin
					fstate <= IDLE;
				end
			endcase
		end
	end
	/*---------fstate output---------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)begin
			SCLK_temp <= 1'b1;
			LDEN      <= 1'b0;
			SHEN      <= 1'b0;
			ACK1      <= 1'b0;
			ACK2      <= 1'b0;
			SDO       <= 1'b1;//SDIN_temp control data[26]/raising/falling
			countEN   <= 1'b0;
			rstcount  <= 1'b0;
			rstACK    <= 1'b0;
		end
		else begin
			case(fstate)
				IDLE:begin
					SCLK_temp <= 1'b1;//high
					LDEN      <= 1'b0;
					SHEN      <= 1'b0;
					ACK1      <= 1'b0;
					ACK2      <= 1'b0;
					SDO       <= 1'b1;//high
					countEN   <= 1'b0;
					rstcount  <= 1'b0;
					rstACK    <= 1'b0;
				end
				GO:begin
					SCLK_temp <= 1'b1;//high
					LDEN      <= 1'b1;//load data
					SHEN      <= 1'b0;
					ACK1      <= 1'b0;
					ACK2      <= 1'b0;
					SDO       <= 1'b1;//high
					countEN   <= 1'b0;
					rstcount  <= 1'b0;
					rstACK    <= 1'b0;
				end
				START:begin
					SCLK_temp <= 1'b1;//high
					LDEN      <= 1'b0;
					SHEN      <= 1'b0;
					ACK1      <= 1'b0;
					ACK2      <= 1'b0;
					SDO       <= 1'b0;//falling(start)
					countEN   <= 1'b0;
					rstcount  <= 1'b0;
					rstACK    <= 1'b0;
				end
				WAIT:begin
					SCLK_temp <= 1'b1;//high
					LDEN      <= 1'b0;
					SHEN      <= 1'b0;
					ACK1      <= 1'b0;
					ACK2      <= 1'b0;
					SDO       <= 1'b0;//low
					countEN   <= 1'b0;
					rstcount  <= 1'b0;
					rstACK    <= 1'b0;
				end
				SHIFT:begin
					SCLK_temp <= 1'b0;//don't care
					LDEN      <= 1'b0;
					SHEN      <= 1'b1;//shifting
					//ACK1
						if(count==5'd7) ACK1 <= 1'b1;
						else            ACK1 <= 1'b0;
					//ACK2
						if(count==5'd16)ACK2 <= 1'b1;
						else            ACK2 <= 1'b0;
					SDO       <= 1'b1;//don't care(data[26])
					countEN   <= 1'b1;//counting
					//rstcount
						if(count==5'd17)rstcount  <= 1'b1;
						else            rstcount  <= 1'b0;
					rstACK    <= 1'b0;
				end
				STOP:begin
					SCLK_temp <= 1'b0;//stop the clock
					LDEN      <= 1'b0;
					SHEN      <= 1'b0;
					ACK1      <= 1'b0;
					ACK2      <= 1'b0;
					SDO       <= 1'b0;//low
					countEN   <= 1'b0;
					rstcount  <= 1'b0;
					rstACK    <= 1'b0;
				end
				FINAL:begin
					SCLK_temp <= 1'b1;//high
					LDEN      <= 1'b0;
					SHEN      <= 1'b0;
					ACK1      <= 1'b0;
					ACK2      <= 1'b0;
					SDO       <= 1'b0;//low
					countEN   <= 1'b0;
					rstcount  <= 1'b0;
					rstACK    <= 1'b0;
				end
				END:begin
					SCLK_temp <= 1'b1;//high
					LDEN      <= 1'b0;
					SHEN      <= 1'b0;
					ACK1      <= 1'b0;
					ACK2      <= 1'b0;
					SDO       <= 1'b1;//raising
					countEN   <= 1'b0;
					rstcount  <= 1'b0;
					rstACK    <= 1'b1;//reset ACK
				end
			endcase
		end
	end
	endmodule
```

## TestBench

```verilog
	`timescale 1ns/1ns
	module tb_I2C_control_18bit();
	reg rst_n;
   reg clk_sys;
   reg en;
   reg [4:0]count;
   wire SCLK_temp;
   wire countEN;
   wire rstcount;
   wire LDEN;
   wire ACK1;
   wire ACK2;
   wire rstACK;
   wire SHEN;
   wire SDO;
	wire SCLK;//add
	
	I2C_control_18bit UUT(
		.rst_n(rst_n),
		.clk_sys(clk_sys),
		.en(en),
		.count(count),
		.SCLK_temp(SCLK_temp),
		.countEN(countEN),
		.rstcount(rstcount),
		.LDEN(LDEN),
		.ACK1(ACK1),
		.ACK2(ACK2),
		.rstACK(rstACK),
		.SHEN(SHEN),
		.SDO(SDO),
		.SCLK(SCLK)
	);
	initial begin
		clk_sys = 1'b0;
	end
	always #50 clk_sys = ~clk_sys;
	initial begin
		rst_n = 1'b0;
		while(1)
		#100 rst_n = 1'b1;
	end
	initial begin
		en = 1'b0;
		#100 en = 1'b1;
		#100 en = 1'b0;
	end
	initial begin
		count = 5'd0;
		#650 count = 5'd1;
		#100 count = 5'd2;
		#100 count = 5'd3;
		#100 count = 5'd4;
		#100 count = 5'd5;
		#100 count = 5'd6;
		#100 count = 5'd7;
		#100 count = 5'd8;
		#100 count = 5'd9;
		#100 count = 5'd10;
		#100 count = 5'd11;
		#100 count = 5'd12;
		#100 count = 5'd13;
		#100 count = 5'd14;
		#100 count = 5'd15;
		#100 count = 5'd16;
		#100 count = 5'd17;
//		#100 count = 5'd18;
//		#100 count = 5'd19;
//		#100 count = 5'd20;
//		#100 count = 5'd21;
//		#100 count = 5'd22;
//		#100 count = 5'd23;
//		#100 count = 5'd24;
//		#100 count = 5'd25;
//		#100 count = 5'd26;
		#1000 $finish;
	end
	initial begin
		$monitor("time=%5d, rst_n=%d, count=%d, ACK1=%d, ACK2=%d",$time, rst_n, count, ACK1, ACK2);
	end
	endmodule
```

## Wave

![I2C_control(18bit)(sol2)(%E6%9C%AA%E5%90%ABtick)%2059d8a15569ca4e7cafce46122798cfe0/Untitled.png](I2C_control(18bit)(sol2)(%E6%9C%AA%E5%90%ABtick)%2059d8a15569ca4e7cafce46122798cfe0/Untitled.png)

## Compilation Report

![I2C_control(18bit)(sol2)(%E6%9C%AA%E5%90%ABtick)%2059d8a15569ca4e7cafce46122798cfe0/Untitled%201.png](I2C_control(18bit)(sol2)(%E6%9C%AA%E5%90%ABtick)%2059d8a15569ca4e7cafce46122798cfe0/Untitled%201.png)

## Monitor

![I2C_control(18bit)(sol2)(%E6%9C%AA%E5%90%ABtick)%2059d8a15569ca4e7cafce46122798cfe0/Untitled%202.png](I2C_control(18bit)(sol2)(%E6%9C%AA%E5%90%ABtick)%2059d8a15569ca4e7cafce46122798cfe0/Untitled%202.png)

## RTL Viewer

![I2C_control(18bit)(sol2)(%E6%9C%AA%E5%90%ABtick)%2059d8a15569ca4e7cafce46122798cfe0/Untitled%203.png](I2C_control(18bit)(sol2)(%E6%9C%AA%E5%90%ABtick)%2059d8a15569ca4e7cafce46122798cfe0/Untitled%203.png)

## State Machine

![I2C_control(27bit)(sol2)(%E6%9C%AA%E5%90%ABtick)%205d66b1fed8234d03a6806dd62b7ac81a/Untitled%204.png](I2C_control(27bit)(sol2)(%E6%9C%AA%E5%90%ABtick)%205d66b1fed8234d03a6806dd62b7ac81a/Untitled%204.png)