# I2C_write(27bit)(未含tick)

## I2C_write

```verilog
	`include "I2C_control.v"
	module I2C_write(clk_sys, rst_n, en, din, ACK, ACK1, ACK2, ACK3, rstACK, SCLK, SDA, ldnACK1, ldnACK2, ldnACK3);
	/*---------ports declaration---------*/
	input       clk_sys, rst_n, en;
	input       [26:0]din;
	output reg  ACK1, ACK2, ACK3;
	output wire ACK, rstACK, SCLK, ldnACK1, ldnACK2, ldnACK3;
	inout       SDA;
	/*---------variables---------*/
	reg  [4:0]count;
	reg  [26:0]regdata;
	wire SCLK_temp, SHEN, LDEN, SDO, countEN, rstcount;
	wire SEL;
	/*---------assign wire---------*/
	assign SEL = (SHEN)?(regdata[26]):(SDO);
	assign SDA = (SEL)?(1'bz):(1'b0);
	assign ACK = ACK1|ACK2|ACK3;
	/*---------module instantiate---------*/
	I2C_control U0(
		.clk_sys(clk_sys),
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
		.SDO(SDO)
	);
	/*---------counter---------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)begin
			count <= 5'd0;
		end
		else begin
			if(rstcount)    count <= 5'd0;
			else if(countEN)count <= count + 5'd1;
			else            count <= count;
		end
	end
	/*---------load data---------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)begin
			regdata <= 27'd0;
		end
		else begin
			if(LDEN)     regdata <= din;
			else if(SHEN)regdata <= {regdata[25:0],1'b0};
			else         regdata <= regdata;
		end
	end
	/*---------ACK---------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)begin
			ACK1 <= 1'b0;
			ACK2 <= 1'b0;
			ACK3 <= 1'b0;
		end
		else if(rstACK)begin
			ACK1 <= 1'b0;
			ACK2 <= 1'b0;
			ACK3 <= 1'b0;
		end
		else begin
			if(ldnACK1)ACK1 <= SDA;
			else       ACK1 <= ACK1;
			if(ldnACK2)ACK2 <= SDA;
			else       ACK2 <= ACK2;
			if(ldnACK3)ACK3 <= SDA;
			else       ACK3 <= ACK3;
		end
	end
	endmodule
```

## TestBench

```verilog
`timescale 1ns/1ns
	module tb_I2C_write();
	
	reg  clk_sys, rst_n, en;
	reg  [26:0]din;
	wire ACK1, ACK2, ACK3;
	wire ACK, SDA, rstACK, SCLK, ldnACK1, ldnACK2, ldnACK3;
	I2C_write UUT(
		.clk_sys(clk_sys),
		.rst_n(rst_n),
		.en(en),
		.din(din), 
		.ACK(ACK),
		.ACK1(ACK1),
		.ACK2(ACK2),
		.ACK3(ACK3),
		.rstACK(rstACK),
		.SCLK(SCLK),
		.SDA(SDA), 
		.ldnACK1(ldnACK1),
		.ldnACK2(ldnACK2),
		.ldnACK3(ldnACK3)
	);
	initial begin
		clk_sys = 0;
		din = 27'b10101010_1_10101010_1_10101010_1;
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
	endmodule
```

## Wave

![I2C_write(27bit)(%E6%9C%AA%E5%90%ABtick)%20dd1f3e31607a49d882769f0a7bb09a38/Untitled.png](I2C_write(27bit)(%E6%9C%AA%E5%90%ABtick)%20dd1f3e31607a49d882769f0a7bb09a38/Untitled.png)

![I2C_write(27bit)(%E6%9C%AA%E5%90%ABtick)%20dd1f3e31607a49d882769f0a7bb09a38/Untitled%201.png](I2C_write(27bit)(%E6%9C%AA%E5%90%ABtick)%20dd1f3e31607a49d882769f0a7bb09a38/Untitled%201.png)

![I2C_write(27bit)(%E6%9C%AA%E5%90%ABtick)%20dd1f3e31607a49d882769f0a7bb09a38/Untitled%202.png](I2C_write(27bit)(%E6%9C%AA%E5%90%ABtick)%20dd1f3e31607a49d882769f0a7bb09a38/Untitled%202.png)

## I2C_control

```verilog
	module I2C_control(clk_sys, rst_n, en, count, countEN, rstcount, ACK1, ACK2, ACK3, rstACK, SCLK, SCLK_temp,  SHEN, LDEN, SDO);
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
	output reg  ACK1, ACK2, ACK3, SCLK_temp, SHEN, LDEN, SDO, countEN, rstcount, rstACK;
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
					if(count==5'd26)fstate <= STOP;
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
			ACK3      <= 1'b0;
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
						if(count==5'd26)rstcount  <= 1'b1;
						else            rstcount  <= 1'b0;
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
	end
	endmodule
```

有修改ACK觸發時間點

8   —→7

17 —→16

26 —→25

## Compilation Report

![I2C_write(27bit)(%E6%9C%AA%E5%90%ABtick)%20dd1f3e31607a49d882769f0a7bb09a38/Untitled%203.png](I2C_write(27bit)(%E6%9C%AA%E5%90%ABtick)%20dd1f3e31607a49d882769f0a7bb09a38/Untitled%203.png)

## RTL Viewer

![I2C_write(27bit)(%E6%9C%AA%E5%90%ABtick)%20dd1f3e31607a49d882769f0a7bb09a38/Untitled%204.png](I2C_write(27bit)(%E6%9C%AA%E5%90%ABtick)%20dd1f3e31607a49d882769f0a7bb09a38/Untitled%204.png)

## State Machine

![I2C_write(27bit)(%E6%9C%AA%E5%90%ABtick)%20dd1f3e31607a49d882769f0a7bb09a38/Untitled%205.png](I2C_write(27bit)(%E6%9C%AA%E5%90%ABtick)%20dd1f3e31607a49d882769f0a7bb09a38/Untitled%205.png)