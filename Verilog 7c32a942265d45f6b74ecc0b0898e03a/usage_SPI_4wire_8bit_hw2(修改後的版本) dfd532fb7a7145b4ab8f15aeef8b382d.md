# usage_SPI_4wire_8bit_hw2(修改後的版本)

## hw2

```verilog
`include "uasge_SPI_4wire_8bit.v"
	module usage_SPI_4wire_8bit_hw2(clk_50M, reset_n, write, write_value, write_complete, read, read_value, read_complete, spi_csn, spi_sck, spi_do, spi_di);
	input  clk_50M, reset_n, write, read, spi_di;
	input  [7:0] write_value;
	output write_complete, read_complete, spi_csn, spi_sck, spi_do;
	output [7:0] read_value;
	uasge_SPI_4wire_8bit U0(
		.clock_50M(clk_50M), 
		.rst_n(reset_n),
		.CS(spi_csn),
		.mosi(spi_do), 
		.miso(spi_di),
		.din(write_value),
		.dout(read_value),
		.SCLK(spi_sck),
		.write(write),
		.read(read),
		.write_complete(write_complete),
		.read_complete(read_complete)
	);
	endmodule
```

## uasge_SPI_4wire_8bit

```verilog
`include "SPI_8bit.v"
	module uasge_SPI_4wire_8bit(clock_50M, rst_n, CS, mosi, miso, din, dout, SCLK, write, read, write_complete, read_complete);
	/*-------ports declaration------*/
	input      clock_50M, rst_n, miso, write, read;
	input      [7:0]din;
	output     CS, mosi, SCLK, write_complete, read_complete;
	output reg [7:0]dout;
	/*-------variables------*/
	reg  [2:0]counter;
	reg  [7:0]regdata_i;
	reg  [7:0]regdata_o;
	reg  tick_SPI, tick_SPI_neg;
	reg  SCLK_temp;//mux of SCLK_temp or 1'b1
	reg  [5:0]cnt_SPI;//1MHZ counter
	wire countEN, rstcount,SHEN, LDEN;
	/*-------assign wire------*/
	assign mosi = (SHEN)?(regdata_i[7]):(1'b0);
	/*-------module instantiate------*/
	SPI_8bit U0(
		.clk_sys(clock_50M),
		.SCLK_temp(SCLK_temp), 
		.rst_n(rst_n),
		.tick_SPI(tick_SPI),
		.SCLK(SCLK),
		.CS(CS),
		.count(counter),
		.countEN(countEN),
		.rstcount(rstcount),
		.SHEN(SHEN),
		.LDEN(LDEN),
		.write(write),
		.read(read),
		.write_complete(write_complete),
		.read_complete(read_complete)
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
	always@(posedge clock_50M or negedge rst_n)begin
		if(!rst_n)tick_SPI_neg <= 1'b0;
		else begin
			if(cnt_SPI==6'd23)tick_SPI_neg <= 1'b1;
			else              tick_SPI_neg <= 1'b0;
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
			counter <= 3'd0;
		end
		else begin
			if(tick_SPI)begin
				if(rstcount)    counter <= 3'd0;
				else if(countEN)counter <= counter + 3'd1;
				else            counter <= 3'd0;
			end
			else counter <= counter;
		end
	end
	/*-------data in------*/
	always@(posedge clock_50M or negedge rst_n)begin//SCLK's negedge
		if(!rst_n)begin
			regdata_i <= 8'd0;
		end
		else begin
			if(tick_SPI)begin
				if(LDEN)     regdata_i <= din;
				else if(SHEN)regdata_i[7:0] <= {regdata_i[6:0], 1'b0};
				else         regdata_i       <= regdata_i;
			end
			else regdata_i <= regdata_i;
		end
	end
	/*-------collect data miso------*/
	always@(posedge clock_50M or negedge rst_n)begin//SCLK's posedge
		if(!rst_n)regdata_o <= 8'd0;
		else begin
			if(tick_SPI_neg)begin//if(tick_SPI)begin
				if(SHEN)regdata_o[7:0] <= {regdata_o[6:0], miso};
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
			dout <= 7'd0;
		end
		else begin
			if(rstcount)dout <= regdata_o;
			else        dout <= dout;
		end
	end
	endmodule
```

## SPI_8bit

```verilog
module SPI_8bit(clk_sys, SCLK_temp, tick_SPI, rst_n, SCLK, CS, count, countEN, rstcount, SHEN, LDEN, write, read, write_complete, read_complete);
	/*-----------parameter-----------*/
	parameter IDLE     = 3'd0;
	parameter START    = 3'd1;
	parameter SHITF    = 3'd2;
	parameter STOP     = 3'd3;
	parameter COMPLETE = 3'd4;
	parameter CHECK    = 3'd5;
	/*-----------in/output-----------*/
	input clk_sys, SCLK_temp, rst_n, tick_SPI, write, read;
	input [2:0]count;
	output reg	CS, countEN, rstcount, SHEN, LDEN, write_complete, read_complete;
	output wire SCLK;
	/*-----------variables-----------*/
	reg [2:0]fstate;
	reg R_W;//write = 0, read = 1
	reg R_W_flag;
	/*-----------assign-----------*/
	assign SCLK = (SHEN)?(~SCLK_temp):(1'b0);
	/*-----------write_complete-----------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)write_complete <= 1'b0;
		else begin
			if(fstate==STOP&&(!R_W)&&tick_SPI) write_complete <= 1'b1;
			else if(fstate==IDLE)              write_complete <= write_complete;
			else                               write_complete <= 1'b0;
		end
	end
	/*-----------read_complete-----------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)read_complete <= 1'b0;
		else begin
			if(fstate==STOP&&(R_W)&&tick_SPI) read_complete <= 1'b1;
			else if(fstate==IDLE)             read_complete <= read_complete;
			else                              read_complete <= 1'b0;
		end
	end
	/*-----------state-----------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)fstate <= IDLE;
		else begin
			case(fstate)
				IDLE:begin
					if(read||write)fstate <= START;
					else          fstate <= IDLE;
				end
				START:begin
					if(tick_SPI)fstate <= SHITF;
					else        fstate <= START;
				end
				SHITF:begin
					if(count==3'd7&&tick_SPI)fstate <= STOP;
					else                     fstate <= SHITF;
				end
				STOP:begin
					if(tick_SPI)fstate <= COMPLETE;
					else        fstate <= STOP;
				end
				COMPLETE:begin
					if(tick_SPI)fstate <= CHECK;
					else        fstate <= COMPLETE;
				end
				CHECK:begin
					if(R_W_flag)fstate <= START;
					else        fstate <= IDLE;
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
			R_W      <= 1'b0;
			R_W_flag <= 1'b0;
		end
		else begin
			case(fstate)
				IDLE:begin
					CS       <= 1'b1;
					countEN  <= 1'b0;
					rstcount <= 1'b0;
					SHEN     <= 1'b0;
					LDEN     <= 1'b0;
					if(write)    R_W <= 1'b0;
					else if(read)R_W <= 1'b1;
					else         R_W <= R_W;
					R_W_flag <= 1'b0;
				end
				START:begin
					CS       <= 1'b0;
					countEN  <= 1'b0;
					rstcount <= 1'b0;
					SHEN     <= 1'b0;
					LDEN     <= 1'b1;
					R_W      <= R_W;
					R_W_flag <= 1'b0;
				end
				SHITF:begin
					CS       <= 1'b0;
					if(tick_SPI)begin
						if(count==3'd7)    rstcount  <= 1'b1;
						else if(count<3'd7)rstcount  <= 1'b0;
						else               rstcount  <= 1'b0;//prevent latch
						SHEN     <= 1'b1;
						countEN  <= 1'b1;
					end
					else begin
						rstcount <= rstcount;
						SHEN     <= SHEN;
						countEN  <= countEN;
					end
					LDEN     <= 1'b0;
					R_W      <= R_W;
					R_W_flag <= R_W_flag;
				end
				STOP:begin
					CS       <= 1'b0;
					countEN  <= 1'b0;
					rstcount <= 1'b0;
					SHEN     <= 1'b0;
					LDEN     <= 1'b0;
					R_W      <= R_W;
					R_W_flag <= R_W_flag;
				end
				COMPLETE:begin
					CS       <= 1'b0;
					countEN  <= 1'b0;
					rstcount <= 1'b0;
					SHEN     <= 1'b0;
					LDEN     <= 1'b0;
					if(write)    R_W <= 1'b0;
					else if(read)R_W <= 1'b1;
					else         R_W <= R_W;
					if(write||read)R_W_flag <= 1'b1;
					else           R_W_flag <= 1'b0;
				end
				CHECK:begin
					CS       <= 1'b0;
					countEN  <= 1'b0;
					rstcount <= 1'b0;
					SHEN     <= 1'b0;
					LDEN     <= 1'b0;
					R_W      <= R_W;
					R_W_flag <= R_W_flag;
				end
				default:begin
					CS       <= 1'b0;
					countEN  <= 1'b0;
					rstcount <= 1'b0;
					SHEN     <= 1'b0;
					LDEN     <= 1'b0;
					R_W      <= 1'b0;
					R_W_flag <= 1'b0;
				end
			endcase
		end
	end
	endmodule
```

## TestBench

```verilog
`timescale 1ns/10ps
	//`timescale 1ns/10ps
	`include "M25AA010A.v"
	module hw2_tb;

	reg     clk_50M;
	reg     reset_n;
	reg     write;
	wire    write_complete;
	//reg   write_complete;
	reg     read;
	wire    read_complete;
	//reg   read_complete;
	reg     [7:0] write_value;
	wire    [7:0] read_value;
	wire    spi_csn;
	wire    spi_sck;
	wire    spi_do;
	wire    spi_di;
	
	usage_SPI_4wire_8bit_hw2 u1 (
		 .clk_50M(clk_50M),
		 .reset_n(reset_n),    
		 .write(write),
		 .write_value(write_value),
		 .write_complete(write_complete),    
		 .read(read),
		 .read_value(read_value),
		 .read_complete(read_complete),   
		 // spi bus
		 .spi_csn(spi_csn),
		 .spi_sck(spi_sck),
		 .spi_do(spi_do),
		 .spi_di(spi_di)
		 );
		 
	M25AA010A u2(
		 .SI(spi_do), 
		 .SO(spi_di), 
		 .SCK(spi_sck), 
		 .CS_N(spi_csn), 
		 .WP_N(1'b1), 
		 .HOLD_N(1'b1), 
		 .RESET(~reset_n)
		 );

	always
	  #10 clk_50M = ~clk_50M;
	  
	initial
	  begin
	  reset_n = 0;    
	  clk_50M = 0 ;
	  write = 0;
	  write_value = 8'h00;
	  read = 0;  
	  #30 reset_n = 1;
	  
	  spi_write(8'h06);  // set write enable
	  
	  #1_000_000;
	  
	  spi_write(8'h02);  // write cmd
	  spi_write(8'h00);  // write addr
	  spi_write(8'h78);  // write data
	  
	  #6_000_000;

	  spi_write(8'h06);  // set write enable
	  
	  #1_000_000;
	  
	  spi_write(8'h02);  // write cmd
	  spi_write(8'h01);  // write addr
	  spi_write(8'h9A);  // write data
	  
	  #6_000_000;   
	  
	  spi_write(8'h06);  // set write enable
	  
	  #1_000_000;
	  
	  spi_write(8'h02);  // write cmd
	  spi_write(8'h02);  // write addr
	  spi_write(8'hBC);  // write data
	  
	  #6_000_000;
	  
	  ////////////////////////////////////////
	  // SPI Read
	  ///////////////////////////////////////
	  
	  #1_000_000;
	  
	  spi_write(8'h03);  // read cmd
	  spi_write(8'h00);  // readd addr
	  spi_read;          // read data

	  #1_000_000;
	  
	  spi_write(8'h03);  // read cmd
	  spi_write(8'h01);  // readd addr
	  spi_read;          // read data  
	  
	  #1_000_000;
	  
	  spi_write(8'h03);  // read cmd
	  spi_write(8'h02);  // readd addr
	  spi_read;          // read data
	  
	  #1_000_000; 
	  $finish;
	  end
	  
	initial
	  begin
	  $monitor("time=%3d memory_00=0x%x memory_01=0x%x memory_02=0x%x",$time,u2.MemoryByte00[7:0],u2.MemoryByte01[7:0],u2.MemoryByte02[7:0]);
	  end

	//initial 
	//  begin  
	//  $fsdbDumpfile("hw2_tb.fsdb");  
	//  $fsdbDumpvars(0,hw2_tb); 
	//  end
	  
	task spi_write; 
	 input [7:0] data; 
	 begin
	  write_value = data;
	  #1_000; // 
	  write = 1;
	  #1_000; // 
	  write = 0;
	  
	  wait(write_complete == 1);
	  $display("time=%3d write_date=0x%x ", $time,write_value);
	 
	 end
	endtask 

	task spi_read;    
	 begin
	  #1_000; // 
	  read = 1;
	  #1_000; // 
	  read = 0;  
	  wait(read_complete == 1);
	  $display("time=%3d read_date=0x%x ", $time,read_value);
	 end
	endtask 
	  
	endmodule
```

## Wave

WREN

![usage_SPI_4wire_8bit_hw2(%E4%BF%AE%E6%94%B9%E5%BE%8C%E7%9A%84%E7%89%88%E6%9C%AC)%20dfd532fb7a7145b4ab8f15aeef8b382d/Untitled.png](usage_SPI_4wire_8bit_hw2(%E4%BF%AE%E6%94%B9%E5%BE%8C%E7%9A%84%E7%89%88%E6%9C%AC)%20dfd532fb7a7145b4ab8f15aeef8b382d/Untitled.png)

vs

![Usage%204%20wire%20SPI%20on%20hw2(%E4%BA%A4%E5%87%BA%E5%8E%BB%E7%9A%84%E7%89%88%E6%9C%AC)%2039a925f4a987472e88e001244690f647/SPI_Write_0X06(WREN).png](Usage%204%20wire%20SPI%20on%20hw2(%E4%BA%A4%E5%87%BA%E5%8E%BB%E7%9A%84%E7%89%88%E6%9C%AC)%2039a925f4a987472e88e001244690f647/SPI_Write_0X06(WREN).png)

////////////////////////////////////////////////////////////////////////////////////////

Write

![usage_SPI_4wire_8bit_hw2(%E4%BF%AE%E6%94%B9%E5%BE%8C%E7%9A%84%E7%89%88%E6%9C%AC)%20dfd532fb7a7145b4ab8f15aeef8b382d/Untitled%201.png](usage_SPI_4wire_8bit_hw2(%E4%BF%AE%E6%94%B9%E5%BE%8C%E7%9A%84%E7%89%88%E6%9C%AC)%20dfd532fb7a7145b4ab8f15aeef8b382d/Untitled%201.png)

vs

![Usage%204%20wire%20SPI%20on%20hw2(%E4%BA%A4%E5%87%BA%E5%8E%BB%E7%9A%84%E7%89%88%E6%9C%AC)%2039a925f4a987472e88e001244690f647/SPI_Write_0X02_0X00_0X78.png](Usage%204%20wire%20SPI%20on%20hw2(%E4%BA%A4%E5%87%BA%E5%8E%BB%E7%9A%84%E7%89%88%E6%9C%AC)%2039a925f4a987472e88e001244690f647/SPI_Write_0X02_0X00_0X78.png)

////////////////////////////////////////////////////////////////////////////////////////

Read

![usage_SPI_4wire_8bit_hw2(%E4%BF%AE%E6%94%B9%E5%BE%8C%E7%9A%84%E7%89%88%E6%9C%AC)%20dfd532fb7a7145b4ab8f15aeef8b382d/Untitled%202.png](usage_SPI_4wire_8bit_hw2(%E4%BF%AE%E6%94%B9%E5%BE%8C%E7%9A%84%E7%89%88%E6%9C%AC)%20dfd532fb7a7145b4ab8f15aeef8b382d/Untitled%202.png)

vs

![Usage%204%20wire%20SPI%20on%20hw2(%E4%BA%A4%E5%87%BA%E5%8E%BB%E7%9A%84%E7%89%88%E6%9C%AC)%2039a925f4a987472e88e001244690f647/SPI_Read_0X03_0X00_0X78.png](Usage%204%20wire%20SPI%20on%20hw2(%E4%BA%A4%E5%87%BA%E5%8E%BB%E7%9A%84%E7%89%88%E6%9C%AC)%2039a925f4a987472e88e001244690f647/SPI_Read_0X03_0X00_0X78.png)

////////////////////////////////////////////////////////////////////////////////////////

All

![usage_SPI_4wire_8bit_hw2(%E4%BF%AE%E6%94%B9%E5%BE%8C%E7%9A%84%E7%89%88%E6%9C%AC)%20dfd532fb7a7145b4ab8f15aeef8b382d/Untitled%203.png](usage_SPI_4wire_8bit_hw2(%E4%BF%AE%E6%94%B9%E5%BE%8C%E7%9A%84%E7%89%88%E6%9C%AC)%20dfd532fb7a7145b4ab8f15aeef8b382d/Untitled%203.png)

![Usage%204%20wire%20SPI%20on%20hw2(%E4%BA%A4%E5%87%BA%E5%8E%BB%E7%9A%84%E7%89%88%E6%9C%AC)%2039a925f4a987472e88e001244690f647.png](Usage%204%20wire%20SPI%20on%20hw2(%E4%BA%A4%E5%87%BA%E5%8E%BB%E7%9A%84%E7%89%88%E6%9C%AC)%2039a925f4a987472e88e001244690f647.png)

////////////////////////////////////////////////////////////////////////////////////////

## monitor

![usage_SPI_4wire_8bit_hw2(%E4%BF%AE%E6%94%B9%E5%BE%8C%E7%9A%84%E7%89%88%E6%9C%AC)%20dfd532fb7a7145b4ab8f15aeef8b382d/Untitled%204.png](usage_SPI_4wire_8bit_hw2(%E4%BF%AE%E6%94%B9%E5%BE%8C%E7%9A%84%E7%89%88%E6%9C%AC)%20dfd532fb7a7145b4ab8f15aeef8b382d/Untitled%204.png)

vs

![Usage%204%20wire%20SPI%20on%20hw2(%E4%BA%A4%E5%87%BA%E5%8E%BB%E7%9A%84%E7%89%88%E6%9C%AC)%2039a925f4a987472e88e001244690f647%201.png](Usage%204%20wire%20SPI%20on%20hw2(%E4%BA%A4%E5%87%BA%E5%8E%BB%E7%9A%84%E7%89%88%E6%9C%AC)%2039a925f4a987472e88e001244690f647%201.png)

## Compilation Report

![usage_SPI_4wire_8bit_hw2(%E4%BF%AE%E6%94%B9%E5%BE%8C%E7%9A%84%E7%89%88%E6%9C%AC)%20dfd532fb7a7145b4ab8f15aeef8b382d/Untitled%205.png](usage_SPI_4wire_8bit_hw2(%E4%BF%AE%E6%94%B9%E5%BE%8C%E7%9A%84%E7%89%88%E6%9C%AC)%20dfd532fb7a7145b4ab8f15aeef8b382d/Untitled%205.png)

## RTL Viewer

![usage_SPI_4wire_8bit_hw2(%E4%BF%AE%E6%94%B9%E5%BE%8C%E7%9A%84%E7%89%88%E6%9C%AC)%20dfd532fb7a7145b4ab8f15aeef8b382d/Untitled%206.png](usage_SPI_4wire_8bit_hw2(%E4%BF%AE%E6%94%B9%E5%BE%8C%E7%9A%84%E7%89%88%E6%9C%AC)%20dfd532fb7a7145b4ab8f15aeef8b382d/Untitled%206.png)

![usage_SPI_4wire_8bit_hw2(%E4%BF%AE%E6%94%B9%E5%BE%8C%E7%9A%84%E7%89%88%E6%9C%AC)%20dfd532fb7a7145b4ab8f15aeef8b382d/Untitled%207.png](usage_SPI_4wire_8bit_hw2(%E4%BF%AE%E6%94%B9%E5%BE%8C%E7%9A%84%E7%89%88%E6%9C%AC)%20dfd532fb7a7145b4ab8f15aeef8b382d/Untitled%207.png)

![usage_SPI_4wire_8bit_hw2(%E4%BF%AE%E6%94%B9%E5%BE%8C%E7%9A%84%E7%89%88%E6%9C%AC)%20dfd532fb7a7145b4ab8f15aeef8b382d/Untitled%208.png](usage_SPI_4wire_8bit_hw2(%E4%BF%AE%E6%94%B9%E5%BE%8C%E7%9A%84%E7%89%88%E6%9C%AC)%20dfd532fb7a7145b4ab8f15aeef8b382d/Untitled%208.png)

## State Machine

![usage_SPI_4wire_8bit_hw2(%E4%BF%AE%E6%94%B9%E5%BE%8C%E7%9A%84%E7%89%88%E6%9C%AC)%20dfd532fb7a7145b4ab8f15aeef8b382d/Untitled%209.png](usage_SPI_4wire_8bit_hw2(%E4%BF%AE%E6%94%B9%E5%BE%8C%E7%9A%84%E7%89%88%E6%9C%AC)%20dfd532fb7a7145b4ab8f15aeef8b382d/Untitled%209.png)