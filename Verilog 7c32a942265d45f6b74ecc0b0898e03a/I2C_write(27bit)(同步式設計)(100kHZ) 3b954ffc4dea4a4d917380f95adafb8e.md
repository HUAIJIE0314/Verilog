# I2C_write(27bit)(同步式設計)(100kHZ)

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
	reg  [4:0] count;
	reg  [8:0] cnt_I2C;
	reg  [26:0]regdata;
	reg        tick_I2C;
	reg        SCLK_100k;
	wire       SCLK_temp, SHEN, LDEN, SDO, countEN, rstcount;
	wire       SEL;
	/*---------assign wire---------*/
	assign SEL = (SHEN)?(regdata[26]):(SDO);
	assign SDA = (SEL)?(1'bz):(1'b0);
	assign ACK = ACK1|ACK2|ACK3;
	/*---------module instantiate---------*/
	I2C_control U0(
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
		.SDO(SDO)
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
	/*---------I2C counter---------*/
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
			regdata <= 27'd0;
		end
		else begin
			if(tick_I2C)begin
				if(LDEN)     regdata <= din;
				else if(SHEN)regdata <= {regdata[25:0],1'b0};
				else         regdata <= regdata;
			end
			else regdata <= regdata;
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
	`include "edge_detect.v"
	`timescale 1ns/1ns
	module tb_I2C_write();
	parameter data1 = 2'd0;
	parameter data2 = 2'd1;
	parameter data3 = 2'd2;
	parameter data4 = 2'd3;
	reg  clk_sys, rst_n, en;
	reg  [26:0]din;
	wire ACK1, ACK2, ACK3;
	wire ACK, rstACK, SCLK, ldnACK1, ldnACK2, ldnACK3;
	wire SDA1, SDA2;
	reg regsda, flag;
	reg [1:0]tb_state;
	reg  [4:0]count;
	wire pos_edge;
	assign SDA1 = (flag)?(regsda):(SDA2);
	integer i;
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
		.SDA(SDA1), 
		.ldnACK1(ldnACK1),
		.ldnACK2(ldnACK2),
		.ldnACK3(ldnACK3)
	);
	initial begin
		regsda   = 1'b0;
		flag     = 1'b0;
		i        = 0;
		tb_state = 2'd0;
		count    = 5'd0;
		clk_sys  = 0;
		din      = 27'b10101010_1_10101010_1_10101010_1;
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
		if(count==5'd28)count <= 5'd0;
		else            count <= count + 5'd1;
		if(count==5'd8)begin
			flag   <= 1'b1;
			regsda <= 1'b0;
		end
		else if(count==5'd17)begin
			flag   <= 1'b1;
			regsda <= 1'b0;
		end
		else if(count==5'd26)begin
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
	
	initial begin
		#8000000 $finish;
	end
	
	
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
				din = 27'b10101010_1_10101010_1_10101010_1;
			end 
			data2:begin
				din = 27'b11001100_1_10010010_1_01011111_1;
			end
			data3:begin
				din = 27'b00110100_1_10100110_1_01011110_1;
			end
			data4:begin
				din = 27'b00001111_1_00110011_1_10011001_1;
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
	/*
	initial begin
		#9000 $stop;
	end
	*/
	endmodule
```

## module Edge_detect

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

/*
always@(posedge clk) 
begin 
  data_in_d1 <=#1 data_in;
  data_in_d2 <=#1 data_in_d1;   
end
 */
endmodule
```

## Wave

![I2C_write(27bit)(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(100kHZ)%203b954ffc4dea4a4d917380f95adafb8e/Untitled.png](I2C_write(27bit)(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(100kHZ)%203b954ffc4dea4a4d917380f95adafb8e/Untitled.png)

![I2C_write(27bit)(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(100kHZ)%203b954ffc4dea4a4d917380f95adafb8e/Untitled%201.png](I2C_write(27bit)(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(100kHZ)%203b954ffc4dea4a4d917380f95adafb8e/Untitled%201.png)

## I2C_control

```verilog
module I2C_control(clk_sys, SCLK_100k, tick_I2C, rst_n, en, count, countEN, rstcount, ACK1, ACK2, ACK3, rstACK, SCLK, SCLK_temp,  SHEN, LDEN, SDO);
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
	input       clk_sys, SCLK_100k, tick_I2C, rst_n, en;
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
					if(count==5'd25&&tick_I2C)fstate <= STOP;
					else                      fstate <= SHIFT;
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
							if(count==5'd25)rstcount  <= 1'b1;
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

有修改ACK觸發時間點

8   —→7

17 —→16

26 —→25

## Compilation Report

![I2C_write(27bit)(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(100kHZ)%203b954ffc4dea4a4d917380f95adafb8e/Untitled%202.png](I2C_write(27bit)(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(100kHZ)%203b954ffc4dea4a4d917380f95adafb8e/Untitled%202.png)

## RTL Viewer

![I2C_write(27bit)(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(100kHZ)%203b954ffc4dea4a4d917380f95adafb8e/Untitled%203.png](I2C_write(27bit)(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(100kHZ)%203b954ffc4dea4a4d917380f95adafb8e/Untitled%203.png)

## State Machine

![I2C_write(27bit)(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(100kHZ)%203b954ffc4dea4a4d917380f95adafb8e/Untitled%204.png](I2C_write(27bit)(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(100kHZ)%203b954ffc4dea4a4d917380f95adafb8e/Untitled%204.png)