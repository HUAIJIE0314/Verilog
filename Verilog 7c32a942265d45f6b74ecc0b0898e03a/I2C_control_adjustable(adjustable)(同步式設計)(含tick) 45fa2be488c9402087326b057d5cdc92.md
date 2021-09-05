# I2C_control_adjustable(adjustable)(同步式設計)(含tick)

!!! 因為結束時不必再持續一個100kHZ所以當count一到25就跳Stop狀態

     而8, 17, 26 改成 7, 16, 25是因為輸出到外面module會delay一個clock

## I2C_control_adjustable

```verilog
module I2C_control_adjustable#(parameter WSize=7'd16, parameter RSize=7'd16)(
	clk_sys, SCLK_500k, tick_I2C, rst_n, en, count, countEN, rstcount, ldnACK, 
	rstACK, SCLK, SCLK_temp,  SHEN, LDEN, SDO, R_W
	);
	/*---------determined data size---------*/
	//		7'd16--> 1 address + 1 data      //
	//		7'd25--> 1 address + 2 data      //
	//		7'd34--> 1 address + 3 data      //
	//		7'd43--> 1 address + 4 data      //
	//		7'd52--> 1 address + 5 data      //
	//		7'd61--> 1 address + 6 data      //
	//		7'd70--> 1 address + 7 data      //
	//		7'd79--> 1 address + 8 data      //
	/*---------parameter---------*/
	localparam IDLE  = 3'd0;
	localparam GO    = 3'd1;
	localparam START = 3'd2;
	localparam WAIT  = 3'd3;
	localparam SHIFT = 3'd4;
	localparam STOP  = 3'd5;
	localparam FINAL = 3'd6;
	localparam END   = 3'd7;
	/*---------ports declaration---------*/
	input       clk_sys, SCLK_500k, tick_I2C, rst_n, en, R_W;
	input       [6:0]count;
	output reg  SCLK_temp, SHEN, LDEN, SDO, countEN, rstcount, rstACK;
	output reg  [8:0]ldnACK;
	output wire SCLK;
	/*---------variables---------*/
	reg  [2:0]fstate;
	wire [6:0]size;
	/*---------assign wire---------*/
	assign SCLK = (SHEN)?(~SCLK_500k):(SCLK_temp);
	assign size = (R_W)?(RSize):(WSize);
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
					if(count==size&&tick_I2C)fstate <= STOP;
					else                     fstate <= SHIFT;
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
			ldnACK    <= 9'd0;
			SDO       <= 1'b1;//SDIN_temp control data/raising/falling
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
						ldnACK    <= 9'd0;
						SDO       <= 1'b1;//high
						countEN   <= 1'b0;
						rstcount  <= 1'b0;
						rstACK    <= 1'b0;
					end
					GO:begin
						SCLK_temp <= 1'b1;//high
						LDEN      <= 1'b1;//load data
						SHEN      <= 1'b0;
						ldnACK    <= 9'd0;
						SDO       <= 1'b1;//high
						countEN   <= 1'b0;
						rstcount  <= 1'b0;
						rstACK    <= 1'b0;
					end
					START:begin
						SCLK_temp <= 1'b1;//high
						LDEN      <= 1'b0;
						SHEN      <= 1'b0;
						ldnACK    <= 9'd0;
						SDO       <= 1'b0;//falling(start)
						countEN   <= 1'b0;
						rstcount  <= 1'b0;
						rstACK    <= 1'b0;
					end
					WAIT:begin
						SCLK_temp <= 1'b1;//high
						LDEN      <= 1'b0;
						SHEN      <= 1'b0;
						ldnACK    <= 9'd0;
						SDO       <= 1'b0;//low
						countEN   <= 1'b0;
						rstcount  <= 1'b0;
						rstACK    <= 1'b0;
					end
					SHIFT:begin
						SCLK_temp <= 1'b0;//don't care
						LDEN      <= 1'b0;
						SHEN      <= 1'b1;//shifting
						//ldnACK
						case(count)
							7'd7: ldnACK <= 9'd256;
							7'd16:ldnACK <= 9'd128;
							7'd25:ldnACK <= 9'd64;
							7'd34:ldnACK <= 9'd32;
							7'd43:ldnACK <= 9'd16;
							7'd52:ldnACK <= 9'd8;
							7'd61:ldnACK <= 9'd4;
							7'd70:ldnACK <= 9'd2;
							7'd79:ldnACK <= 9'd1;
							default:ldnACK <= 9'd0;
						endcase
						SDO       <= 1'b1;//don't care(data[26])
						countEN   <= 1'b1;//counting
						//rstcount
							if(count==size)rstcount  <= 1'b1;
							else           rstcount  <= 1'b0;
						rstACK    <= 1'b0;
					end
					STOP:begin
						SCLK_temp <= 1'b0;//stop the clock
						LDEN      <= 1'b0;
						SHEN      <= 1'b0;
						ldnACK    <= 9'd0;
						SDO       <= 1'b0;//low
						countEN   <= 1'b0;
						rstcount  <= 1'b0;
						rstACK    <= 1'b0;
					end
					FINAL:begin
						SCLK_temp <= 1'b1;//high
						LDEN      <= 1'b0;
						SHEN      <= 1'b0;
						ldnACK    <= 9'd0;
						SDO       <= 1'b0;//low
						countEN   <= 1'b0;
						rstcount  <= 1'b0;
						rstACK    <= 1'b0;
					end
					END:begin
						SCLK_temp <= 1'b1;//high
						LDEN      <= 1'b0;
						SHEN      <= 1'b0;
						ldnACK    <= 9'd0;
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
				ldnACK    <= ldnACK;
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
	module tb_I2C_control_adjustable();
	
	reg  rst_n;
   reg  clk_sys;
   reg  en;
   reg  [6:0]count;
	reg  tick_I2C;
	reg  SCLK_500k;
	reg  R_W;
   wire SCLK_temp;
   wire countEN;
   wire rstcount;
   wire LDEN;
   wire [8:0]ldnACK;
   wire rstACK;
   wire SHEN;
   wire SDO;
	wire SCLK;//add
	reg  [6:0]cnt_I2C;
	reg  [2:0]size;
	wire pos_edge;

	edge_detect U0(
		.clk(clk_sys),
		.rst_n(rst_n),
		.data_in(rstACK),
		.pos_edge(pos_edge),
		.neg_edge ()
   );
	/*
		7'd16--> 1 address + 1 data
		7'd25--> 1 address + 2 data
		7'd34--> 1 address + 3 data
		7'd43--> 1 address + 4 data
		7'd52--> 1 address + 5 data
		7'd61--> 1 address + 6 data
		7'd70--> 1 address + 7 data
		7'd79--> 1 address + 8 data
	*/
	/*
	defparam I2C.WSize = 7'd79;
	defparam I2C.RSize = 7'd79;
	*/
	I2C_control_adjustable#(.WSize(61), .RSize(79)) I2C(
		.rst_n(rst_n),
		.clk_sys(clk_sys),
		.SCLK_500k(SCLK_500k), 
		.tick_I2C(tick_I2C),
		.en(en),
		.count(count),
		.SCLK_temp(SCLK_temp),
		.countEN(countEN),
		.rstcount(rstcount),
		.LDEN(LDEN),
		.ldnACK(ldnACK),
		.rstACK(rstACK),
		.SHEN(SHEN),
		.SDO(SDO),
		.SCLK(SCLK),
		.R_W(R_W)
	);
	/*---------500k counter---------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)cnt_I2C <= 7'd0;
		else begin
			if(cnt_I2C<7'd99)cnt_I2C <= cnt_I2C + 7'd1;//0-99
			else             cnt_I2C <= 7'd0;
		end
	end
	/*---------I2C tick---------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)tick_I2C <= 1'b0;
		else begin
			if(cnt_I2C==7'd98)tick_I2C <= 1'b1;
			else              tick_I2C <= 1'b0;
		end
	end
	/*---------SCLK_500k---------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)SCLK_500k <= 1'b0;
		else begin
			if(cnt_I2C==7'd49)     SCLK_500k <= 1'b0;
			else if(cnt_I2C==7'd99)SCLK_500k <= 1'b1;
			else                   SCLK_500k <= SCLK_500k;
		end
	end
	initial begin
		clk_sys = 1'b0;
		size    = 3'd0;
	end
	always #50 clk_sys = ~clk_sys;
	initial begin
		rst_n = 1'b0;
		while(1)
		#100 rst_n = 1'b1;
	end
	initial begin
		R_W = 0;
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
		#1000 $finish;
	end
	*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)count = 7'd0;
		else begin
			if(tick_I2C&&SHEN)count <= count + 7'd1;
			else if(rstcount) count <= 7'd0;
			else              count <= count;
		end
	end
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)size <= 3'd0;
		else begin
			if(pos_edge)begin
				if(size<3'd1)begin
					R_W <= 1;
					size <= size + 3'd1;
					#100 en = 1'b1;
					#100 en = 1'b0;
				end
			end
		end
	end
	initial begin
		$monitor("time=%5d, rst_n=%d, count=%d",$time, rst_n, count);
	end
	initial begin
		#8550050 $finish;
	end
	endmodule
```

## Wave

![I2C_control_adjustable(adjustable)(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABtick)%2045fa2be488c9402087326b057d5cdc92/Untitled.png](I2C_control_adjustable(adjustable)(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABtick)%2045fa2be488c9402087326b057d5cdc92/Untitled.png)

![I2C_control_adjustable(adjustable)(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABtick)%2045fa2be488c9402087326b057d5cdc92/Untitled%201.png](I2C_control_adjustable(adjustable)(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABtick)%2045fa2be488c9402087326b057d5cdc92/Untitled%201.png)

![I2C_control_adjustable(adjustable)(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABtick)%2045fa2be488c9402087326b057d5cdc92/Untitled%202.png](I2C_control_adjustable(adjustable)(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABtick)%2045fa2be488c9402087326b057d5cdc92/Untitled%202.png)

![I2C_control_adjustable(adjustable)(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABtick)%2045fa2be488c9402087326b057d5cdc92/Untitled%203.png](I2C_control_adjustable(adjustable)(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABtick)%2045fa2be488c9402087326b057d5cdc92/Untitled%203.png)

## Compilation Report

![I2C_control_adjustable(adjustable)(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABtick)%2045fa2be488c9402087326b057d5cdc92/Untitled%204.png](I2C_control_adjustable(adjustable)(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABtick)%2045fa2be488c9402087326b057d5cdc92/Untitled%204.png)

## RTL Viewer

![I2C_control_adjustable(adjustable)(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABtick)%2045fa2be488c9402087326b057d5cdc92/Untitled%205.png](I2C_control_adjustable(adjustable)(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABtick)%2045fa2be488c9402087326b057d5cdc92/Untitled%205.png)

## State Machine

![I2C_control_adjustable(adjustable)(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABtick)%2045fa2be488c9402087326b057d5cdc92/Untitled%206.png](I2C_control_adjustable(adjustable)(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABtick)%2045fa2be488c9402087326b057d5cdc92/Untitled%206.png)