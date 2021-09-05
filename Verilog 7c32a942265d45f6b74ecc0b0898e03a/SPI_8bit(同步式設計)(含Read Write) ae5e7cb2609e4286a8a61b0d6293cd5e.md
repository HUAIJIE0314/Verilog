# SPI_8bit(同步式設計)(含Read/Write)

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

---

## TestBench

```verilog
	`timescale 1ns/1ns
	module tb_SPI();
	
	parameter delay = 20;
	
	reg clk_sys, SCLK_temp, tick_SPI, rst_n, write, read;
	reg [2:0]count;
	wire CS, countEN, rstcount, SHEN, LDEN;
	wire SCLK, write_complete, read_complete;
	//wire [2:0]fs;
	
	reg [5:0]cnt_SPI;
	
	
	reg [3:0]count_time;
	SPI_8bit U0(
		.clk_sys(clk_sys),
		.SCLK_temp(SCLK_temp), 
		.tick_SPI(tick_SPI),
		.rst_n(rst_n), 
		.SCLK(SCLK), 
		.CS(CS), 
		.count(count), 
		.countEN(countEN), 
		.rstcount(rstcount), 
		.SHEN(SHEN), 
		.LDEN(LDEN),
		.write(write),
		.read(read),
		.write_complete(write_complete), 
		.read_complete(read_complete)
		//.fs(fs)
	);
	
	initial begin
		count_time = 4'd0;
		clk_sys = 0;
		SCLK_temp = 0;
   end
	initial begin
		while(1)
			#(delay/2) clk_sys = ~clk_sys;
	end
	initial begin
       rst_n = 0;
       while(1)
           #100 rst_n = 1;
   end
	initial begin
       write = 0;
		 read  = 0; 
       #100 write = 1;
       #100 write = 0;
		 
		 #1000_000;
		 spi_write();
		 spi_write();
		 spi_write();
		 spi_write();
		 spi_write();
		 spi_read();
		 spi_read();
		 spi_read();
		 spi_read();
		 spi_read();
		 #1000_000 $finish;
		 
   end
	/*-------1M counter------*/
	always@(posedge clk_sys or negedge rst_n)begin
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
	/*-------1M tick------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)tick_SPI <= 1'b0;
		else begin
			if(cnt_SPI==6'd48)tick_SPI <= 1'b1;
			else              tick_SPI <= 1'b0;
		end
	end
	/*-------ctrl counter------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)begin
			count <= 3'd0;
		end
		else begin
			if(tick_SPI)begin
				if(rstcount)    count <= 3'd0;
				else if(countEN)count <= count + 3'd1;
				else            count <= 3'd0;
			end
			else count <= count;
		end
	end
	/*-------make SCLK------*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)begin
			SCLK_temp <= 1'b0;
		end
		else begin
			if(cnt_SPI == 6'd24)     SCLK_temp <= 1'b0;
			else if(cnt_SPI == 6'd49)SCLK_temp <= 1'b1;
			else                     SCLK_temp <= SCLK_temp;
		end
	end
	/*
	always@(posedge clk_sys or negedge rst_n)begin
		if((read_complete|write_complete))begin 
			if(count_time<4'd15)count_time <= count_time + 4'd1;
			else                count_time <= count_time;
			if(count_time<4'd5)begin
				write <= 1;
			end
			else if(count_time<4'd10)begin
				read <= 1;
			end
			else begin
				read  <= 0;
				write <= 0;
			end
		end
		else begin
			write <= 0;
			read <= 0;
		end
	end
	*/
	/*
	always@(posedge write_complete or posedge read_complete)begin
		if(count_time<4'd15)count_time <= count_time + 4'd1;
		else                count_time <= count_time;
		if(count_time<4'd5)begin
				write <= 1;
				#500 write <= 0;
		end
		else if(count_time<4'd10)begin
				read <= 1;
				#500 read <= 0;
		end
		else begin
				read  <= 0;
				write <= 0;
		end
	end
	*/
	
	task spi_write; 
	 begin
	  #1000; // 
	  write = 1;
	  #1000; // 
	  write = 0;
	  wait(write_complete == 1);
	 end
	endtask 

	task spi_read;    
	 begin
	  #1000; // 
	  read = 1;
	  #1000; // 
	  read = 0;  
	  wait(read_complete == 1);
	 end
	endtask 
	

	endmodule
```

## Wave

![SPI_8bit(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABRead%20Write)%20ae5e7cb2609e4286a8a61b0d6293cd5e/Untitled.png](SPI_8bit(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABRead%20Write)%20ae5e7cb2609e4286a8a61b0d6293cd5e/Untitled.png)

## Compilation Report

![SPI_8bit(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABRead%20Write)%20ae5e7cb2609e4286a8a61b0d6293cd5e/Untitled%201.png](SPI_8bit(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABRead%20Write)%20ae5e7cb2609e4286a8a61b0d6293cd5e/Untitled%201.png)

## RTL Viewer

![SPI_8bit(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABRead%20Write)%20ae5e7cb2609e4286a8a61b0d6293cd5e/Untitled%202.png](SPI_8bit(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABRead%20Write)%20ae5e7cb2609e4286a8a61b0d6293cd5e/Untitled%202.png)

## State Machine

![SPI_8bit(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABRead%20Write)%20ae5e7cb2609e4286a8a61b0d6293cd5e/Untitled%203.png](SPI_8bit(%E5%90%8C%E6%AD%A5%E5%BC%8F%E8%A8%AD%E8%A8%88)(%E5%90%ABRead%20Write)%20ae5e7cb2609e4286a8a61b0d6293cd5e/Untitled%203.png)