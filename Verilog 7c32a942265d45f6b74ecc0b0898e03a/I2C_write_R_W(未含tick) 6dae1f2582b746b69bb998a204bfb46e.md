# I2C_write_R_W(未含tick)

## I2C_write_R_W

```verilog
	module I2C_write_R_W(clk_sys, rst_n, en, data_R, data_W, ACK, ACK1, ACK2, ACK3, rstACK, SCLK, SDA, ldnACK1, ldnACK2, ldnACK3, R_W);
	/*---------ports declaration---------*/
	input       clk_sys, rst_n, en, R_W;
	input       [26:0]data_R;//read  27bit
	input       [17:0]data_W;//write 18bit
	output reg  ACK1, ACK2, ACK3;
	output wire ACK, rstACK, SCLK, ldnACK1, ldnACK2, ldnACK3;
	inout       SDA;
	/*---------variables---------*/
	reg  [4:0]count;
	reg  [26:0]regdata_R;
	reg  [17:0]regdata_W;
	wire SCLK_temp, SHEN, LDEN, SDO, countEN, rstcount;
	wire SEL;
	/*---------assign wire---------*/
	//assign SEL = (SHEN)?(regdata[26]):(SDO);
	assign SEL = (SHEN)?((!R_W)?(regdata_W[17]):(regdata_R[26])):(SDO);
	assign SDA = (SEL)?(1'bz):(1'b0);
	assign ACK = ACK1|ACK2|ACK3;
	/*---------module instantiate---------*/
	I2C_control_R_W U0(
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
		.SDO(SDO),
		.R_W(R_W)
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
			regdata_R <= 27'd0;
			regdata_W <= 18'd0;
		end
		else begin
			if(!R_W)begin
				if(LDEN)     regdata_W <= data_W;
				else if(SHEN)regdata_W <= {regdata_W[16:0],1'b0};
				else         regdata_W <= regdata_W;
			end 
			else begin
				if(LDEN)     regdata_R <= data_R;
				else if(SHEN)regdata_R <= {regdata_R[25:0],1'b0};
				else         regdata_R <= regdata_R;
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
	module tb_I2C_write_R_W();
	
	reg  clk_sys, rst_n, en, R_W;
	reg  [26:0]data_R;
	reg  [17:0]data_W;
	wire ACK1, ACK2, ACK3;
	wire ACK, rstACK, SCLK, ldnACK1, ldnACK2, ldnACK3;
	wire SDA1, SDA2;
	reg regsda, flag;
	reg  [4:0]count;
	assign SDA1 = (flag)?(regsda):(SDA2);
	
	I2C_write_R_W UUT(
		.clk_sys(clk_sys),
		.rst_n(rst_n),
		.en(en),
		.data_R(data_R), 
		.data_W(data_W),
		.ACK(ACK),
		.ACK1(ACK1),
		.ACK2(ACK2),
		.ACK3(ACK3),
		.rstACK(rstACK),
		.SCLK(SCLK),
		.SDA(SDA1), 
		.ldnACK1(ldnACK1),
		.ldnACK2(ldnACK2),
		.ldnACK3(ldnACK3),
		.R_W(R_W)
	);
	initial begin
		regsda  = 1'b0;
		flag    = 1'b0;
		count   = 5'd0;
		clk_sys = 1'b0;
		data_W  = {8'h34, 1'b1, 8'h96, 1'b1};
		//data_R  = 27'b10101010_1_10101010_1_10101010_1;
		data_R  = 27'd0;
		R_W     = 1'b0;
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
			if(count==5'd19)count <= 5'd0;
			else            count <= count + 5'd1;
		end
		else begin
			if(count==5'd28)count <= 5'd0;
			else            count <= count + 5'd1;
		end
		
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
		#2800 R_W = 1'b1;
		#100  data_R = {8'h55, 1'b1, 8'h62, 1'b1, 8'h23, 1'b1};
		      data_W = 18'd0;
		#100  en = 1;
		#100  en = 0;
	end
	initial begin
		#10000 $stop;
	end
	endmodule
```

## Wave

![I2C_write_R_W(%E6%9C%AA%E5%90%ABtick)%206dae1f2582b746b69bb998a204bfb46e/Untitled.png](I2C_write_R_W(%E6%9C%AA%E5%90%ABtick)%206dae1f2582b746b69bb998a204bfb46e/Untitled.png)

## Compitation Repot

![I2C_write_R_W(%E6%9C%AA%E5%90%ABtick)%206dae1f2582b746b69bb998a204bfb46e/Untitled%201.png](I2C_write_R_W(%E6%9C%AA%E5%90%ABtick)%206dae1f2582b746b69bb998a204bfb46e/Untitled%201.png)

## RTL Viewer

![I2C_write_R_W(%E6%9C%AA%E5%90%ABtick)%206dae1f2582b746b69bb998a204bfb46e/Untitled%202.png](I2C_write_R_W(%E6%9C%AA%E5%90%ABtick)%206dae1f2582b746b69bb998a204bfb46e/Untitled%202.png)

## State Machine

![I2C_write_R_W(%E6%9C%AA%E5%90%ABtick)%206dae1f2582b746b69bb998a204bfb46e/Untitled%203.png](I2C_write_R_W(%E6%9C%AA%E5%90%ABtick)%206dae1f2582b746b69bb998a204bfb46e/Untitled%203.png)