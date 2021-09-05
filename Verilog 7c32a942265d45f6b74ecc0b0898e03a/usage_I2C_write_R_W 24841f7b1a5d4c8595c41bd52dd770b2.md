# usage_I2C_write_R_W

## usage_I2C_write_R_W

```verilog
`include "I2C_write_R_W.v"
	module usage_I2C_write_R_W(en, clk_sys, rst_n, addr, data_i, SCLK, SDA, data_o);
	/*------parameter------*/
	parameter IDLE  = 2'd0;
	parameter SHIFT = 2'd1;
	parameter STOP  = 2'd2;
	/*------ports declaration------*/
	input      clk_sys, rst_n, en;
	input [7:0]addr, data_i;
	inout      SDA;
	output     SCLK;
	output reg [15:0]data_o;
	/*------variables------*/
	reg  [15:0]data_temp;
	reg  [1:0]fstate;
	reg  [3:0]count;
	wire ACK, rstACK, tick_I2C_neg;
	wire ldnACK1, ldnACK2, ldnACK3;
	/*------module I2C_write instantiate------*/
	I2C_write_R_W U0(
		.clk_sys(clk_sys),
		.rst_n(rst_n),
		.en(en), 
		.data_R({addr, 1'b1, 8'hff, 1'b0, 8'hff, 1'b1}),//27bit
		.data_W({addr, 1'b1, data_i, 1'b1}),//18bit
		.ACK(ACK),
		.ACK1(),
		.ACK2(),
		.ACK3(), 
		.rstACK(rstACK),
		.SCLK(SCLK),
		.SDA(SDA),
		.ldnACK1(ldnACK1), 
		.ldnACK2(ldnACK2),
		.ldnACK3(ldnACK3),
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
					if(ldnACK1&&(addr[0])&&tick_I2C_neg)fstate <= SHIFT;
					else                            fstate <= IDLE;
				end
				SHIFT:begin
					if(count==4'd15&&tick_I2C_neg)fstate <= STOP;
					else            fstate <= SHIFT;
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
					if(!ldnACK2&&tick_I2C_neg)begin
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
					count     <= 4'dx;
					data_temp <= 16'hxx;
					data_o    <= 16'hxx;
				end
			endcase
		end
	end
	endmodule
```

## I2C_write_R_W

```verilog
`include "I2C_control_R_W.v"
	module I2C_write_R_W(clk_sys, rst_n, en, data_R, data_W, ACK, ACK1, ACK2, ACK3, rstACK, SCLK, SDA, ldnACK1, ldnACK2, ldnACK3, R_W, tick_I2C_neg);
	/*---------ports declaration---------*/
	input       clk_sys, rst_n, en, R_W;
	input       [26:0]data_R;//read  27bit
	input       [17:0]data_W;//write 18bit
	output reg  ACK1, ACK2, ACK3;
	output wire ACK, rstACK, SCLK, ldnACK1, ldnACK2, ldnACK3;
	inout       SDA;
	/*---------variables---------*/
	reg  [4:0] count;
	reg  [8:0] cnt_I2C;
	reg        tick_I2C;
	output reg        tick_I2C_neg;
	reg        SCLK_100k;
	reg  [26:0]regdata_R;
	reg  [17:0]regdata_W;
	wire       SCLK_temp, SHEN, LDEN, SDO, countEN, rstcount;
	wire       SEL;
	/*---------assign wire---------*/
	//assign SEL = (SHEN)?(regdata[26]):(SDO);
	assign SEL = (SHEN)?((!R_W)?(regdata_W[17]):(regdata_R[26])):(SDO);
	assign SDA = (SEL)?(1'bz):(1'b0);
	assign ACK = ACK1|ACK2|ACK3;
	/*---------module instantiate---------*/
	I2C_control_R_W U0(
		.clk_sys(clk_sys),
		.SCLK_100k(SCLK_100k),
		.tick_I2C(tick_I2C),
		.rst_n(rst_n),
		.en(en),
		.count(count),
		.countEN(countEN),
		.rstcount(rstcount),
		.ACK1(ldnACK1),
		.ACK2(ldnACK2),
		.ACK3(ldnACK3),
		.rstACK(rstACK),
		.SCLK(SCLK),
		.SCLK_temp(SCLK_temp),
		.SHEN(SHEN),
		.LDEN(LDEN),
		.SDO(SDO),
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
	/*---------I2C tick_neg---------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)tick_I2C_neg <= 1'b0;
		else begin
			if(cnt_I2C==9'd249)tick_I2C_neg <= 1'b1;
			else               tick_I2C_neg <= 1'b0;
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
	/*---------counter---------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)begin
			count <= 5'd0;
		end
		else begin
			if(tick_I2C)begin
				if(rstcount)    count <= 5'd0;
				else if(countEN)count <= count + 5'd1;
				else            count <= count;
			end
			else count <= count;
		end
	end
	/*---------load data---------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)begin
			regdata_R <= 27'd0;
			regdata_W <= 18'd0;
		end
		else begin
			if(!R_W)begin
				if(tick_I2C)begin
					if(LDEN)     regdata_W <= data_W;
					else if(SHEN)regdata_W <= {regdata_W[16:0],1'b0};
					else         regdata_W <= regdata_W;
				end
				else regdata_W <= regdata_W;
			end 
			else begin
				if(tick_I2C)begin
					if(LDEN)     regdata_R <= data_R;
					else if(SHEN)regdata_R <= {regdata_R[25:0],1'b0};
					else         regdata_R <= regdata_R;
				end
				else regdata_R <= regdata_R;
			end
		end
	end
	/*---------ACK---------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)begin
			ACK1 <= 1'b0;
			ACK2 <= 1'b0;
			ACK3 <= 1'b0;
		end
		else if(rstACK&&tick_I2C)begin
			ACK1 <= 1'b0;
			ACK2 <= 1'b0;
			ACK3 <= 1'b0;
		end
		else begin
			if(ldnACK1&&tick_I2C)ACK1 <= SDA;
			else                 ACK1 <= ACK1;
			if(ldnACK2&&tick_I2C)ACK2 <= SDA;
			else                 ACK2 <= ACK2;
			if(ldnACK3&&tick_I2C)ACK3 <= SDA;
			else                 ACK3 <= ACK3;
		end
	end
	endmodule
```

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
	usage_I2C_write_R_W UUT(
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

![usage_I2C_write_R_W%2024841f7b1a5d4c8595c41bd52dd770b2/Untitled.png](usage_I2C_write_R_W%2024841f7b1a5d4c8595c41bd52dd770b2/Untitled.png)

![usage_I2C_write_R_W%2024841f7b1a5d4c8595c41bd52dd770b2/Untitled%201.png](usage_I2C_write_R_W%2024841f7b1a5d4c8595c41bd52dd770b2/Untitled%201.png)

## Compitation Repot

![usage_I2C_write_R_W%2024841f7b1a5d4c8595c41bd52dd770b2/Untitled%202.png](usage_I2C_write_R_W%2024841f7b1a5d4c8595c41bd52dd770b2/Untitled%202.png)

## RTL Viewer

![usage_I2C_write_R_W%2024841f7b1a5d4c8595c41bd52dd770b2/Untitled%203.png](usage_I2C_write_R_W%2024841f7b1a5d4c8595c41bd52dd770b2/Untitled%203.png)

![usage_I2C_write_R_W%2024841f7b1a5d4c8595c41bd52dd770b2/Untitled%204.png](usage_I2C_write_R_W%2024841f7b1a5d4c8595c41bd52dd770b2/Untitled%204.png)

![usage_I2C_write_R_W%2024841f7b1a5d4c8595c41bd52dd770b2/Untitled%205.png](usage_I2C_write_R_W%2024841f7b1a5d4c8595c41bd52dd770b2/Untitled%205.png)

## State Machine

![usage_I2C_write_R_W%2024841f7b1a5d4c8595c41bd52dd770b2/Untitled%206.png](usage_I2C_write_R_W%2024841f7b1a5d4c8595c41bd52dd770b2/Untitled%206.png)

![usage_I2C_write_R_W%2024841f7b1a5d4c8595c41bd52dd770b2/Untitled%207.png](usage_I2C_write_R_W%2024841f7b1a5d4c8595c41bd52dd770b2/Untitled%207.png)