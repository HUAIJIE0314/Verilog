# I2C_control_R_W(同步式設計)(含tick)

!!! 因為結束時不必再持續一個100kHZ所以當count一到25就跳Stop狀態

     而8, 17, 26 改成 7, 16, 25是因為輸出到外面module會delay一個clock

## I2C_control_R_W

```verilog
	module I2C_control_R_W(clk_sys, SCLK_100k, tick_I2C, rst_n, en, count, countEN, rstcount, ACK1, ACK2, ACK3, rstACK, SCLK, SCLK_temp,  SHEN, LDEN, SDO, R_W);
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
	input       clk_sys, rst_n, en, SCLK_100k, tick_I2C, R_W;//W---->0, R---->1
	input       [4:0]count;
	output reg  ACK1, ACK2, ACK3, SCLK_temp, SHEN, LDEN, SDO, countEN, rstcount, rstACK;
	output wire SCLK;
	/*---------variables---------*/
	reg [2:0]fstate;
	/*---------assign wire---------*/
	assign SCLK = (SHEN)?(~SCLK_100k):(SCLK_temp);
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
					if(tick_I2C)fstate <= START;
					else        fstate <= GO;
				end
				START:begin
					if(tick_I2C)fstate <= WAIT;
					else        fstate <= START;
				end
				WAIT:begin
					if(tick_I2C)fstate <= SHIFT;
					else        fstate <= WAIT;
				end
				SHIFT:begin
					if(!R_W)begin//write-->18bit
						if(count==5'd16&&tick_I2C)fstate <= STOP;
						else                      fstate <= SHIFT;
					end
					else begin//read-->27bit
						if(count==5'd25&&tick_I2C)fstate <= STOP;
						else                      fstate <= SHIFT;
					end
				end
				STOP:begin
					if(tick_I2C)fstate <= FINAL;
					else        fstate <= STOP;
				end
				FINAL:begin
					if(tick_I2C)fstate <= END;
					else        fstate <= FINAL;
				end
				END:begin
					if(tick_I2C)fstate <= IDLE;
					else        fstate <= END;
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
			ACK3      <= 1'b0;
			SDO       <= 1'b1;//SDIN_temp control data[26]/raising/falling
			countEN   <= 1'b0;
			rstcount  <= 1'b0;
			rstACK    <= 1'b0;
		end
		else begin
			if(tick_I2C)begin
				case(fstate)
					IDLE:begin
						SCLK_temp <= 1'b1;//high
						LDEN      <= 1'b0;
						SHEN      <= 1'b0;
						ACK1      <= 1'b0;
						ACK2      <= 1'b0;
						ACK3      <= 1'b0;
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
						ACK3      <= 1'b0;
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
						ACK3      <= 1'b0;
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
						ACK3      <= 1'b0;
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
						//ACK3
							if(count==5'd25)ACK3 <= 1'b1;
							else            ACK3 <= 1'b0;
						SDO       <= 1'b1;//don't care(data[26])
						countEN   <= 1'b1;//counting
						//rstcount
							if(!R_W)begin//write-->18bit
								if(count==5'd16)rstcount  <= 1'b1;
								else            rstcount  <= 1'b0;
							end
							else begin//read-->27bit
								if(count==5'd25)rstcount  <= 1'b1;
								else            rstcount  <= 1'b0;
							end
						rstACK    <= 1'b0;
					end
					STOP:begin
						SCLK_temp <= 1'b0;//stop the clock
						LDEN      <= 1'b0;
						SHEN      <= 1'b0;
						ACK1      <= 1'b0;
						ACK2      <= 1'b0;
						ACK3      <= 1'b0;
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
						ACK3      <= 1'b0;
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
						ACK3      <= 1'b0;
						SDO       <= 1'b1;//raising
						countEN   <= 1'b0;
						rstcount  <= 1'b0;
						rstACK    <= 1'b1;//reset ACK
					end
				endcase
			end
			else begin
				SCLK_temp <= SCLK_temp;
				LDEN      <= LDEN;
				SHEN      <= SHEN;
				ACK1      <= ACK1;
				ACK2      <= ACK2;
				ACK3      <= ACK3;
				SDO       <= SDO;
				countEN   <= countEN;
				rstcount  <= rstcount;
				rstACK    <= rstACK;
			end
		end
	end
	endmodule
```

## TestBench

```verilog
	`include "edge_detect.v"
	`timescale 1ns/1ns
	module tb_I2C_control_R_W();
	
	reg  rst_n;
	reg  clk_sys;
	reg  en;
	reg  [4:0]count;
	reg  tick_I2C;
	reg  SCLK_100k;
	reg  R_W;
	wire SCLK_temp;
	wire countEN;
	wire rstcount;
	wire LDEN;
	wire ACK1;
	wire ACK2;
	wire ACK3;
	wire rstACK;
	wire SHEN;
	wire SDO;
	wire SCLK;//add
	reg  [8:0] cnt_I2C;
	reg  flag;
	wire pos_edge;
	I2C_control_R_W UUT(
		.rst_n(rst_n),
		.clk_sys(clk_sys),
		.SCLK_100k(SCLK_100k), 
		.tick_I2C(tick_I2C),
		.en(en),
		.count(count),
		.SCLK_temp(SCLK_temp),
		.countEN(countEN),
		.rstcount(rstcount),
		.LDEN(LDEN),
		.ACK1(ACK1),
		.ACK2(ACK2),
		.ACK3(ACK3),
		.rstACK(rstACK),
		.SHEN(SHEN),
		.SDO(SDO),
		.SCLK(SCLK),
		.R_W(R_W)
	);
	/*---------100k counter---------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)cnt_I2C <= 9'd0;
		else begin
			if(cnt_I2C<9'd499)cnt_I2C <= cnt_I2C + 9'd1;//0-499
			else              cnt_I2C <= 9'd0;
		end
	end
	/*---------I2C tick---------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)tick_I2C <= 1'b0;
		else begin
			if(cnt_I2C==9'd498)tick_I2C <= 1'b1;
			else               tick_I2C <= 1'b0;
		end
	end
	/*---------SCLK_100k---------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)SCLK_100k <= 1'b0;
		else begin
			if(cnt_I2C==9'd249)     SCLK_100k <= 1'b0;
			else if(cnt_I2C==9'd499)SCLK_100k <= 1'b1;
			else                    SCLK_100k <= SCLK_100k;
		end
	end
	initial R_W = 1'b0;//write 18bit
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
	   flag = 0;
		en = 1'b0;
		#100 en = 1'b1;
		#100 en = 1'b0;
	end
	/*
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
		#100 count = 5'd18;
		#100 count = 5'd19;
		#100 count = 5'd20;
		#100 count = 5'd21;
		#100 count = 5'd22;
		#100 count = 5'd23;
		#100 count = 5'd24;
		#100 count = 5'd25;
		#100 count = 5'd26;
		
		R_W      = 1'b1;//write 18bit
		#100 en  = 1'b1;
		#100 en  = 1'b0;
		
		#100 count = 5'd0;
		#100 count = 5'd1;
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
		#100 count = 5'd18;
		#100 count = 5'd19;
		#100 count = 5'd20;
		#100 count = 5'd21;
		#100 count = 5'd22;
		#100 count = 5'd23;
		#100 count = 5'd24;
		#100 count = 5'd25;
		#100 count = 5'd26;
		#1000 $finish;
	end
	*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)count = 5'd0;
		else begin
			if(!R_W)begin
				if(count==5'd18)  count <= 5'd0;
				else              count = count;
				if(tick_I2C&&SHEN)count = count + 5'd1;
				else              count = count;           	
			end
			else begin
				if(count==5'd27)  count <= 5'd0;
				else              count = count;
				if(tick_I2C&&SHEN)count = count + 5'd1;
				else              count = count;
			end
		end
	end
	always@(posedge clk_sys)begin
		if(pos_edge&&!flag)begin
			en   <= 1'b1;
			flag <= 1'b1;
			R_W  <= 1'b1;
		end 
		else en <= 1'b0;
	end
	edge_detect U0(
		.clk(clk_sys),
		.rst_n(rst_n),
		.data_in(rstACK),
		.pos_edge(pos_edge),
		.neg_edge() 
   );
	initial begin
		$monitor("time=%5d, rst_n=%d, count=%d, ACK1=%d, ACK2=%d, ACK3=%d",$time, rst_n, count, ACK1, ACK2, ACK3);
	end
	initial begin
		#3000000 $finish;
	end
	endmodule
```

## Wave

![I2C_control_R_W(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABtick)%20c5bfbe723cc844b8babcb36a28e30669/Untitled.png](I2C_control_R_W(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABtick)%20c5bfbe723cc844b8babcb36a28e30669/Untitled.png)

## Compilation Report

![I2C_control_R_W(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABtick)%20c5bfbe723cc844b8babcb36a28e30669/Untitled%201.png](I2C_control_R_W(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABtick)%20c5bfbe723cc844b8babcb36a28e30669/Untitled%201.png)

## RTL Viewer

![I2C_control_R_W(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABtick)%20c5bfbe723cc844b8babcb36a28e30669/Untitled%202.png](I2C_control_R_W(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABtick)%20c5bfbe723cc844b8babcb36a28e30669/Untitled%202.png)

## State Machine

![I2C_control_R_W(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABtick)%20c5bfbe723cc844b8babcb36a28e30669/Untitled%203.png](I2C_control_R_W(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABtick)%20c5bfbe723cc844b8babcb36a28e30669/Untitled%203.png)