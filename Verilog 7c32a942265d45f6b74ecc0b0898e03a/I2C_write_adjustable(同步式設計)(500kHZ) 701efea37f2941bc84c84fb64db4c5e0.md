# I2C_write_adjustable(同步式設計)(500kHZ)

## I2C_write_adjustable

```verilog
`include "I2C_control_adjustable.v"
	module I2C_write_adjustable#(parameter WSize = 7'd16, parameter RSize = 7'd16)(
	clk_sys, rst_n, en, data_W, data_R, totalACK, ACK, rstACK, SCLK, SDA, ldnACK, R_W, tick_I2C_neg);
	/*---------determined data size---------*/
	//		7'd16--> 1 address + 1 data      //
	//		7'd25--> 1 address + 2 data      //
	//		7'd34--> 1 address + 3 data      //
	//		7'd43--> 1 address + 4 data      //
	//		7'd52--> 1 address + 5 data      //
	//		7'd61--> 1 address + 6 data      //
	//		7'd70--> 1 address + 7 data      //
	//		7'd79--> 1 address + 8 data      //
	/*---------ports declaration---------*/
	input                  clk_sys, rst_n, en, R_W;
	input       [WSize+1:0]data_W;//read
	input       [RSize+1:0]data_R;//write
	output reg        [8:0]ACK;
	output wire       [8:0]ldnACK;
	output reg             tick_I2C_neg;
	output wire            totalACK, rstACK, SCLK;
	inout                  SDA;
	/*---------variables---------*/
	reg       [6:0]count;
	reg       [6:0]cnt_I2C;
	reg [WSize+1:0]regdata_W;
	reg [RSize+1:0]regdata_R;
	reg            tick_I2C;
	reg            SCLK_500k;
	wire           SCLK_temp, SHEN, LDEN, SDO, countEN, rstcount;
	wire           SEL;
	/*---------assign wire---------*/
	assign SEL = (SHEN)?((!R_W)?(regdata_W[WSize+1]):(regdata_R[RSize+1])):(SDO);
	assign SDA = (SEL)?(1'bz):(1'b0);
	assign totalACK = |ACK;
	/*---------module instantiate---------*/
	defparam I2C.WSize = WSize;
	defparam I2C.RSize = RSize;
	I2C_control_adjustable I2C(
		.clk_sys(clk_sys),
		.SCLK_500k(SCLK_500k),
		.tick_I2C(tick_I2C),
		.rst_n(rst_n),
		.en(en),
		.count(count),
		.countEN(countEN),
		.rstcount(rstcount),
		.ldnACK(ldnACK),
		.rstACK(rstACK),
		.SCLK(SCLK),
		.SCLK_temp(SCLK_temp),
		.SHEN(SHEN),
		.LDEN(LDEN),
		.SDO(SDO),
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
	/*---------I2C tick_neg---------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)tick_I2C_neg <= 1'b0;
		else begin
			if(cnt_I2C==7'd48)tick_I2C_neg <= 1'b1;
			else              tick_I2C_neg <= 1'b0;
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
	/*---------counter---------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)begin
			count <= 7'd0;
		end
		else begin
			if(tick_I2C)begin
				if(rstcount)    count <= 7'd0;
				else if(countEN)count <= count + 7'd1;
				else            count <= count;
			end
			else count <= count;
		end
	end
	/*---------load data---------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)begin
			regdata_R <= 'd0;
			regdata_W <= 'd0;
		end
		else begin
			if(!R_W)begin
				if(tick_I2C)begin
					if(LDEN)     regdata_W <= data_W;
					else if(SHEN)regdata_W <= {regdata_W[WSize:0],1'b0};//left rotate
					else         regdata_W <= regdata_W;
				end
				else regdata_W <= regdata_W;
			end 
			else begin
				if(tick_I2C)begin
					if(LDEN)     regdata_R <= data_R;
					else if(SHEN)regdata_R <= {regdata_R[RSize:0],1'b0};//left rotate
					else         regdata_R <= regdata_R;
				end
				else regdata_R <= regdata_R;
			end
		end
	end
	/*---------ACK---------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)ACK  <= 9'd0;
		else if(rstACK&&tick_I2C)ACK  <= 9'd0;
		else begin
			if(ldnACK[8]&&tick_I2C)ACK[8] <= SDA;
			else                   ACK[8] <= ACK[8];
			if(ldnACK[7]&&tick_I2C)ACK[7] <= SDA;
			else                   ACK[7] <= ACK[7];
			if(ldnACK[6]&&tick_I2C)ACK[6] <= SDA;
			else                   ACK[6] <= ACK[6];
			if(ldnACK[5]&&tick_I2C)ACK[5] <= SDA;
			else                   ACK[5] <= ACK[5];
			if(ldnACK[4]&&tick_I2C)ACK[4] <= SDA;
			else                   ACK[4] <= ACK[4];
			if(ldnACK[3]&&tick_I2C)ACK[3] <= SDA;
			else                   ACK[3] <= ACK[3];
			if(ldnACK[2]&&tick_I2C)ACK[2] <= SDA;
			else                   ACK[2] <= ACK[2];
			if(ldnACK[1]&&tick_I2C)ACK[1] <= SDA;
			else                   ACK[1] <= ACK[1];
			if(ldnACK[0]&&tick_I2C)ACK[0] <= SDA;
			else                   ACK[0] <= ACK[0];
		end
	end
	endmodule
```

## TestBench

```verilog
`include "edge_detect.v"
	`timescale 1ns/1ns
	module tb_I2C_write_adjustable();
	parameter data1 = 2'd0;
	parameter data2 = 2'd1;
	parameter data3 = 2'd2;
	parameter data4 = 2'd3;
	reg  clk_sys, rst_n, en, R_W;
	reg  [26:0]data_R;
	reg  [17:0]data_W;
	wire totalACK, rstACK, SCLK;
	wire [8:0]ACK;
	wire [8:0]ldnACK;
	wire SDA1, SDA2;
	reg  regsda, flag;
	reg  [6:0]count;
	reg  [1:0]tb_state;
	wire pos_edge, tick_I2C_neg;
	assign SDA1 = (flag)?(regsda):(SDA2);
	integer i;
	/*---------determined data size---------*/
	//		7'd16--> 1 address + 1 data      //
	//		7'd25--> 1 address + 2 data      //
	//		7'd34--> 1 address + 3 data      //
	//		7'd43--> 1 address + 4 data      //
	//		7'd52--> 1 address + 5 data      //
	//		7'd61--> 1 address + 6 data      //
	//		7'd70--> 1 address + 7 data      //
	//		7'd79--> 1 address + 8 data      //
	//instantiate module I2C
	localparam WSize = 16;
	localparam RSize = 25;
	defparam I2C.WSize = WSize;
	defparam I2C.RSize = RSize;
	I2C_write_adjustable I2C(
		.clk_sys(clk_sys),
		.rst_n(rst_n),
		.en(en), 
		.data_W(data_W),
		.data_R(data_R),
		.totalACK(totalACK),
		.ACK(ACK), 
		.rstACK(rstACK),
		.SCLK(SCLK),
		.SDA(SDA1), 
		.ldnACK(ldnACK), 
		.R_W(R_W),
		.tick_I2C_neg(tick_I2C_neg)
	);
	initial begin
		i        = 0; 
		regsda   = 1'b0;
		flag     = 1'b0;
		tb_state = 2'd0;
		count    = 7'd0;
		clk_sys  = 1'b0;
		data_W   = {8'h34, 1'b1, 8'h96, 1'b1};
		//data_R  = 27'b10101010_1_10101010_1_10101010_1;
		data_R   = 27'd0;
		R_W      = 1'b0;
	end
	always #50 clk_sys = ~clk_sys;
	initial begin
		rst_n = 0;
		while(1)
		#100 rst_n = 1;
	end
	initial begin
		en = 0;
		#100 en = 1;
		#100 en = 0;
	end
	always@(negedge SCLK)begin
		if(!R_W)begin
			if(count==(WSize+2))count <= 7'd0;
			else            count <= count + 7'd1;
		end
		else begin
			if(count==(RSize+2))count <= 7'd0;
			else            count <= count + 7'd1;
		end
		if(count==7'd8)begin
			flag   <= 1'b1;
			regsda <= 1'b0;
		end
		else if(count==7'd17)begin
			flag   <= 1'b1;
			regsda <= 1'b0;
		end
		else if(count==7'd26)begin
			flag   <= 1'b1;
			regsda <= 1'b0;
		end
		else if(count==7'd35)begin
			flag   <= 1'b1;
			regsda <= 1'b0;
		end
		else if(count==7'd44)begin
			flag   <= 1'b1;
			regsda <= 1'b0;
		end
		else if(count==7'd53)begin
			flag   <= 1'b1;
			regsda <= 1'b0;
		end
		else if(count==7'd62)begin
			flag   <= 1'b1;
			regsda <= 1'b0;
		end
		else if(count==7'd71)begin
			flag   <= 1'b1;
			regsda <= 1'b0;
		end
		else if(count==7'd80)begin
			flag   <= 1'b1;
			regsda <= 1'b0;
		end
		else begin
			flag   <= 1'b0;
			regsda <= 1'b1;
		end 
	end
	initial begin
		$monitor("time=%d, SDA=%d, SCLK=%d, count=%d",$time,SDA1,SCLK,count);
	end
	/*
	initial begin
		#2800 R_W = 1'b1;
		#100  data_R = {8'h55, 1'b1, 8'h62, 1'b1, 8'h23, 1'b1};
		      data_W = 18'd0;
		#100  en = 1;
		#100  en = 0;
	end
	initial begin
		#10000 $stop;
	end*/
	always@(negedge clk_sys)begin
		case(tb_state)
			data1:begin
				if(pos_edge)tb_state <= data2;
				else        tb_state <= data1;
			end
			data2:begin
				if(pos_edge)tb_state <= data3;
				else        tb_state <= data2;
			end
			data3:begin
				if(pos_edge)tb_state <= data4;
				else        tb_state <= data3;
			end
			data4:begin
				tb_state <= data4;
			end
		endcase
	end
	always@(posedge clk_sys)begin
		case(tb_state)
			data1:begin
				R_W      = 1'b0;
				data_W   = {8'h34, 1'b1, 8'h96, 1'b1};
				data_R   = 27'd0;
			end 
			data2:begin
				R_W      = 1'b1;
				data_W   = 18'd0;
				data_R   = {8'h99, 1'b1, 8'h28, 1'b1, 8'h13, 1'b1};
			end
			data3:begin
				R_W      = 1'b0;
				data_W   = {8'h61, 1'b1, 8'h89, 1'b1};
				data_R   = 27'd0;
			end
			data4:begin
				R_W      = 1'b1;
				data_W   = 18'd0;
				data_R   = {8'h77, 1'b1, 8'h15, 1'b1, 8'h36, 1'b1};
				i = i + 1;
			end
		endcase
	end
	always@(posedge clk_sys)begin
		if(pos_edge&&(i<=1))en = 1;
		else                en = 0;
	end
	edge_detect U0(
		.clk(clk_sys),
		.rst_n(rst_n),
		.data_in(rstACK),
		.pos_edge(pos_edge),
		.neg_edge()
   );
	initial begin
		#6500000 $finish;
	end
	endmodule
```

## Wave

![I2C_write_adjustable(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(500kHZ)%20701efea37f2941bc84c84fb64db4c5e0/Untitled.png](I2C_write_adjustable(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(500kHZ)%20701efea37f2941bc84c84fb64db4c5e0/Untitled.png)

![I2C_write_adjustable(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(500kHZ)%20701efea37f2941bc84c84fb64db4c5e0/Untitled%201.png](I2C_write_adjustable(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(500kHZ)%20701efea37f2941bc84c84fb64db4c5e0/Untitled%201.png)

## Compitation Repot

![I2C_write_adjustable(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(500kHZ)%20701efea37f2941bc84c84fb64db4c5e0/Untitled%202.png](I2C_write_adjustable(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(500kHZ)%20701efea37f2941bc84c84fb64db4c5e0/Untitled%202.png)

## RTL Viewer

![I2C_write_adjustable(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(500kHZ)%20701efea37f2941bc84c84fb64db4c5e0/Untitled%203.png](I2C_write_adjustable(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(500kHZ)%20701efea37f2941bc84c84fb64db4c5e0/Untitled%203.png)

![I2C_write_adjustable(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(500kHZ)%20701efea37f2941bc84c84fb64db4c5e0/Untitled%204.png](I2C_write_adjustable(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(500kHZ)%20701efea37f2941bc84c84fb64db4c5e0/Untitled%204.png)

## State Machine

![I2C_write_adjustable(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(500kHZ)%20701efea37f2941bc84c84fb64db4c5e0/Untitled%205.png](I2C_write_adjustable(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(500kHZ)%20701efea37f2941bc84c84fb64db4c5e0/Untitled%205.png)