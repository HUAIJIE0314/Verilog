# uasge_SPI_4wire(1MHZ)

## uasge_SPI_4wire

```verilog
	`include "SPI.v"
	module uasge_SPI_4wire(en, clock_50M, rst_n, CS, mosi, miso, din, dout, SCLK, finish);
	/*-------ports declaration------*/
	input      clock_50M, rst_n, miso, en;
	input      [15:0]din;
	output     CS, mosi, SCLK, finish;
	output reg [15:0]dout;
	/*-------variables------*/
	reg  [3:0]counter;
	reg  [15:0]regdata_i;
	reg  [15:0]regdata_o;
	reg  tick_SPI;
	reg  SCLK_temp;//mux of SCLK_temp or 1'b1
	reg  [5:0]cnt_SPI;//1MHZ counter
	wire ready;
	wire countEN, rstcount,SHEN, LDEN;
	/*-------assign wire------*/
	assign mosi = (SHEN)?(regdata_i[15]):(1'b0);
	assign ready = en;
	/*-------module instantiate------*/
	SPI U0(
		.clk_sys(clock_50M),
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
	/*-------1M counter------*/
	always@(posedge clock_50M or negedge rst_n)begin
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
	/*-------SCLK tick posedge and negedge------*/
	always@(posedge clock_50M or negedge rst_n)begin
		if(!rst_n)tick_SPI <= 1'b0;
		else begin
			if(cnt_SPI==6'd48)tick_SPI <= 1'b1;
			else              tick_SPI <= 1'b0;
		end
	end
	/*-------make SCLK------*/
	always@(posedge clock_50M or negedge rst_n)begin
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
	always@(posedge clock_50M or negedge rst_n)begin
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
	/*-------data in------*/
	always@(posedge clock_50M or negedge rst_n)begin//SCLK's negedge
		if(!rst_n)begin
			regdata_i <= 16'd0;
		end
		else begin
			if(tick_SPI)begin
				if(LDEN)     regdata_i <= din;
				else if(SHEN)regdata_i[15:0] <= {regdata_i[14:0], 1'b0};
				else         regdata_i       <= regdata_i;
			end
			else regdata_i <= regdata_i;
		end
	end
	/*-------collect data miso------*/
	always@(posedge clock_50M or negedge rst_n)begin//SCLK's posedge
		if(!rst_n)regdata_o <= 16'd0;
		else begin
			if(tick_SPI)begin
				if(SHEN)regdata_o[15:0] <= {regdata_o[14:0], miso};
				else regdata_o <= regdata_o;			
			end
			else begin
				regdata_o <= regdata_o;
			end 
		end
	end
	/*-------data out------*/
	always@(posedge clock_50M or negedge rst_n)begin
		if(!rst_n)begin
			dout <= 16'd0;
		end
		else begin
			if(finish)dout <= regdata_o;
			else      dout <= dout;
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

## TestBench

```verilog
`timescale 1ns / 1ns

module spi_tb();
	reg  sysClk, rst_n, tick_tx, miso;
	reg  [7:0]address, txdata;
	reg  [15:0]rxdata;
	wire [15:0]data_get;
	wire sclk, cs_n, mosi;

	//wire ready;
	//wire [1:0]st;
	//wire [4:0]rwc;
	//wire tick_rx;
	wire finish;
	localparam Freq_i = 50000000;
	localparam durTime = 1000;
	integer i;
	/*
spi#(.Freq(Freq_i))
	UUT(  .sysClk(sysClk),
			.rst_n(rst_n),
			.tick_tx(tick_tx),
			.tick_rx(tick_rx),
			.addr_i(address),
			.data_i(txdata),
			.data_o(data_get),
			.sclk(sclk),
			.cs_n(cs_n),
			.mosi(mosi),
			.miso(miso),
			.ready(ready),
			.st(st),
			.rwc(rwc));
	*/
	uasge_SPI_4wire UUT(
		.en(tick_tx),
		.clock_50M(sysClk),
		.rst_n(rst_n),
		.CS(cs_n),
		.mosi(mosi),
		.miso(miso),
		.din({address,txdata}),
		.dout(data_get),
		.SCLK(sclk),
		.finish(finish)
	);
	always@(posedge sclk, negedge rst_n)begin
		if(!rst_n)				miso <= 1'b0;
		else if(i>=0&&i<16)  miso <= rxdata[15-i];
		else						miso <= 1'b0;
	end
	
	always@(posedge sclk, negedge rst_n)begin
		if(!rst_n)i <= 0;
		else		 i <= i + 1;
	end
	
	initial forever #1 sysClk = ~sysClk;
	initial begin
		sysClk  = 0;
		rst_n   = 1;
		tick_tx = 0;
		address = 8'h7A;
		txdata  = 8'h81;
		rxdata  = 16'h4689;
	#5 rst_n   = 0;
	#5 rst_n   = 1;
	
	#1 tick_tx = 1;
	#1 tick_tx = 0;
		
	repeat(10) #durTime;
		$stop;
	end

endmodule
```

## Wave

![uasge_SPI_4wire(1MHZ)%20a1a2ee800e9742379d2f95a74ed12aa3/Untitled.png](uasge_SPI_4wire(1MHZ)%20a1a2ee800e9742379d2f95a74ed12aa3/Untitled.png)

## Compilation Report

![uasge_SPI_4wire(1MHZ)%20a1a2ee800e9742379d2f95a74ed12aa3/Untitled%201.png](uasge_SPI_4wire(1MHZ)%20a1a2ee800e9742379d2f95a74ed12aa3/Untitled%201.png)

## RTL Viewer

![uasge_SPI_4wire(1MHZ)%20a1a2ee800e9742379d2f95a74ed12aa3/Untitled%202.png](uasge_SPI_4wire(1MHZ)%20a1a2ee800e9742379d2f95a74ed12aa3/Untitled%202.png)

## State Machine

![uasge_SPI_4wire(1MHZ)%20a1a2ee800e9742379d2f95a74ed12aa3/Untitled%203.png](uasge_SPI_4wire(1MHZ)%20a1a2ee800e9742379d2f95a74ed12aa3/Untitled%203.png)