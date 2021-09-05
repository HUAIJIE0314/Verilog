# usage_I2C_write_adjustable

## usage_I2C_write_adjustable

```verilog
`include "I2C_write_adjustable.v"
	module usage_I2C_write_adjustable(en, clk_sys, rst_n, addr, data_i, SCLK, SDA, data_o);
	/*------parameter------*/
	parameter IDLE  = 2'd0;
	parameter SHIFT = 2'd1;
	parameter STOP  = 2'd2;
	/*------ports declaration------*/
	input            clk_sys, rst_n, en;
	input       [7:0]addr, data_i;
	inout            SDA;
	output           SCLK;
	output reg [15:0]data_o;
	/*------variables------*/
	reg [15:0]data_temp;
	reg  [1:0]fstate;
	reg  [3:0]count;
	wire [8:0]ldnACK;
	wire      ACK, rstACK, tick_I2C_neg;
	/*------instantiate I2C_write_adjustable------*/
	defparam I2C.WSize = 7'd16;
	defparam I2C.RSize = 7'd25;
	/*---------determined data size---------*/
	/////////////////////////////////////////
	//		7'd16--> 1 address + 1 data      //
	//		7'd25--> 1 address + 2 data      //
	//		7'd34--> 1 address + 3 data      //
	//		7'd43--> 1 address + 4 data      //
	//		7'd52--> 1 address + 5 data      //
	//		7'd61--> 1 address + 6 data      //
	//		7'd70--> 1 address + 7 data      //
	//		7'd79--> 1 address + 8 data      //
	/////////////////////////////////////////
	I2C_write_adjustable I2C(
		.clk_sys(clk_sys),
		.rst_n(rst_n),
		.en(en), 
		.data_W({addr, 1'b1, data_i, 1'b1}),
		.data_R({addr, 1'b1, 8'hff, 1'b0, 8'hff, 1'b1}),
		.totalACK(ACK),
		.ACK(), 
		.rstACK(rstACK),
		.SCLK(SCLK),
		.SDA(SDA), 
		.ldnACK(ldnACK), 
		.R_W(addr[0]),
		.tick_I2C_neg(tick_I2C_neg)
	);
	/*------fstate state------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)begin
			fstate <= 2'd0;
		end
		else begin
			case(fstate)
				IDLE:begin
					if(ldnACK[8]&&(addr[0])&&tick_I2C_neg)fstate <= SHIFT;
					else                                  fstate <= IDLE;
				end
				SHIFT:begin
					if(count==4'd15&&tick_I2C_neg)fstate <= STOP;
					else                          fstate <= SHIFT;
				end
				STOP:begin
					fstate <= IDLE;
				end
				default:begin
					fstate <= IDLE;
				end
			endcase
		end
	end
	/*------fstate output------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)begin
			count     <= 4'd0;
			data_temp <= 16'd0;
			data_o    <= 16'd0;
		end
		else begin
			case(fstate)
				IDLE:begin
					count     <= 4'd0;
					data_temp <= data_temp;
				end
				SHIFT:begin
					if(!ldnACK[7]&&tick_I2C_neg)begin
						count     <= count + 4'd1;
						data_temp <= {data_temp[14:0], SDA};
					end 
					else begin
						count     <= count;
						data_temp <= data_temp;
					end
				end
				STOP:begin
					count  <= 4'd0;
					data_o <= data_temp;
				end
				default:begin
					count     <= 4'd0;
					data_temp <= 16'h00;
					data_o    <= 16'h00;
				end
			endcase
		end
	end
	endmodule
```

## I2C_write_adjustable

```verilog
`include "I2C_control_adjustable.v"
	module I2C_write_adjustable#(parameter WSize = 7'd16, parameter RSize = 7'd16)(
	clk_sys, rst_n, en, data_W, data_R, totalACK, ACK, rstACK, SCLK, SDA, ldnACK, R_W, tick_I2C_neg);
	/*---------determined data size---------*/
	/////////////////////////////////////////
	//		7'd16--> 1 address + 1 data      //
	//		7'd25--> 1 address + 2 data      //
	//		7'd34--> 1 address + 3 data      //
	//		7'd43--> 1 address + 4 data      //
	//		7'd52--> 1 address + 5 data      //
	//		7'd61--> 1 address + 6 data      //
	//		7'd70--> 1 address + 7 data      //
	//		7'd79--> 1 address + 8 data      //
	/////////////////////////////////////////
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

## I2C_control_adjustable

```verilog
module I2C_control_adjustable#(parameter WSize=7'd16, parameter RSize=7'd16)(
	clk_sys, SCLK_500k, tick_I2C, rst_n, en, count, countEN, rstcount, ldnACK, 
	rstACK, SCLK, SCLK_temp,  SHEN, LDEN, SDO, R_W
	);
	/*---------determined data size---------*/
	/////////////////////////////////////////
	//		7'd16--> 1 address + 1 data      //
	//		7'd25--> 1 address + 2 data      //
	//		7'd34--> 1 address + 3 data      //
	//		7'd43--> 1 address + 4 data      //
	//		7'd52--> 1 address + 5 data      //
	//		7'd61--> 1 address + 6 data      //
	//		7'd70--> 1 address + 7 data      //
	//		7'd79--> 1 address + 8 data      //
	/////////////////////////////////////////
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
`timescale 1ns / 1ns

module i2c_tb();
	reg sysClk, rst_n, tick_tx, sda_i;
	reg [7:0]addr_i, data_i;
	wire [15:0]data_o;	
	wire tick_rx, SCL, ready;
	wire SDA, sda_o;
	wire txdCont;
	wire [3:0]st;
	wire [3:0]btc;
	wire [2:0]wttime, rdtime;
	localparam Freq_i = 50000000;
	localparam durTime = 5*1000;
	integer i;
	assign sda_o = (sda_i==1'b1)?SDA:sda_i;
	assign SDA   = (sda_i==1'b1)?1'bz:1'b0;
	wire [8:0]ldnACK;
	/*
i2c#(.Freq(Freq_i), .readPacket(3'd2), .writePacket(3'd1))
	UUT(	.sysClk(sysClk), 
			.rst_n(rst_n), 
			.tick_tx(tick_tx), 
			.tick_rx(tick_rx), 
			.txdCont(txdCont),
			.addr_i(addr_i), 
			.data_i(data_i), 
			.data_o(data_o), 
			.SDA(SDA), 
			.SCL(SCL), 
			.ready(ready), 
			.st(st), 
			.btc(btc),
			.wttime(wttime),
			.rdtime(rdtime));
	*/
	usage_I2C_write_adjustable UUT(
		.en(tick_tx), 
		.clk_sys(sysClk), 
		.rst_n(rst_n), 
		.addr(addr_i),
		.data_i(data_i),
		.SCLK(SCL), 
		.SDA(SDA),
		.data_o(data_o)
	);
	always@(negedge SCL, negedge rst_n)begin
		if(!rst_n)		sda_i <= 1'b1;
		else if(i==8||i==17||i==27||i==28||i==36||i==35||i==43)sda_i <= 1'b0;//<27 for write
		else				sda_i <= 1'b1;
	end
	
	always@(posedge SCL, negedge rst_n)begin
		if(!rst_n)i <= 0;
		else		 i <= i + 1;
	end
	
	initial forever #1 sysClk = ~sysClk;
	initial begin
		sysClk  = 0;
		rst_n   = 1;
		tick_tx = 0;
		sda_i   = 1'b1;
		addr_i  = 8'hA6;
		data_i  = 8'h81;
	#50 rst_n   = 0;
	#50 rst_n   = 1;
	
	#1 tick_tx = 1;
	#1 tick_tx = 0;
		
	repeat(5) #durTime;
	
		addr_i  = 8'hA7;
	#1 tick_tx = 1;
	#1 tick_tx = 0;
	
	repeat(10) #durTime;
	
		$stop;
	end

endmodule
```

## Wave

![usage_I2C_write_adjustable%20687495acd0384f3ba8d287e3dbc83f52/Untitled.png](usage_I2C_write_adjustable%20687495acd0384f3ba8d287e3dbc83f52/Untitled.png)

![usage_I2C_write_adjustable%20687495acd0384f3ba8d287e3dbc83f52/Untitled%201.png](usage_I2C_write_adjustable%20687495acd0384f3ba8d287e3dbc83f52/Untitled%201.png)

![usage_I2C_write_adjustable%20687495acd0384f3ba8d287e3dbc83f52/Untitled%202.png](usage_I2C_write_adjustable%20687495acd0384f3ba8d287e3dbc83f52/Untitled%202.png)

## Compitation Repot

![usage_I2C_write_adjustable%20687495acd0384f3ba8d287e3dbc83f52/Untitled%203.png](usage_I2C_write_adjustable%20687495acd0384f3ba8d287e3dbc83f52/Untitled%203.png)

## RTL Viewer

![usage_I2C_write_adjustable%20687495acd0384f3ba8d287e3dbc83f52/Untitled%204.png](usage_I2C_write_adjustable%20687495acd0384f3ba8d287e3dbc83f52/Untitled%204.png)

![usage_I2C_write_adjustable%20687495acd0384f3ba8d287e3dbc83f52/Untitled%205.png](usage_I2C_write_adjustable%20687495acd0384f3ba8d287e3dbc83f52/Untitled%205.png)

![usage_I2C_write_adjustable%20687495acd0384f3ba8d287e3dbc83f52/Untitled%206.png](usage_I2C_write_adjustable%20687495acd0384f3ba8d287e3dbc83f52/Untitled%206.png)

## State Machine

![usage_I2C_write_R_W%2024841f7b1a5d4c8595c41bd52dd770b2/Untitled%206.png](usage_I2C_write_R_W%2024841f7b1a5d4c8595c41bd52dd770b2/Untitled%206.png)

![usage_I2C_write_R_W%2024841f7b1a5d4c8595c41bd52dd770b2/Untitled%207.png](usage_I2C_write_R_W%2024841f7b1a5d4c8595c41bd52dd770b2/Untitled%207.png)

usage_I2C_write_adjustable