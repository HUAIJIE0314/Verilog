# hw3_perfect

## hw3

```verilog
module hw3(
		rst_in_n, 
		clk, 
		sfr_wr, 
		sfr_rd, 
		sfr_addr, 
		sfr_data_out, 
		sfr_data_in, 
		i2c_sda, 
		i2c_scl
	);
	/*---------parameter---------*/
	localparam  IDLE    = 3'd0;
	localparam  GO      = 3'd1;
	localparam  START   = 3'd2;
	localparam  WAIT    = 3'd3;
	localparam  SHIFT   = 3'd4;
	localparam  ACK     = 3'd5;
	localparam  STOP    = 3'd6;
	localparam  FINISH  = 3'd7;
	/*---------ports declaration---------*/
	input       rst_in_n;
	input	      clk; 
	input	      sfr_wr; 
	input	      sfr_rd; 
	input	 [7:0]sfr_addr; 
	input	 [7:0]sfr_data_out; 
	inout       i2c_sda;
	output [7:0]sfr_data_in; 
	output      i2c_scl;
	reg    [7:0]sfr_data_in; 
	/*---------variables---------*/
	reg    [2:0]fstate;
	reg    [8:0]cnt_100k;
	reg    [2:0]index;
	reg    [1:0]times;
	reg    [7:0]regData;
	reg         tick_i2C;
	reg         en;
	reg         clk_100k;
	reg         shift;
	reg         sclTemp;
	reg         sdaTemp;
	wire        sdaMUX;
	/*---------assign wire---------*/
	assign sdaMUX  = (shift)?(regData[7]):(sdaTemp);
	assign i2c_sda = (sdaMUX)?(1'bz):(1'b0);
	assign i2c_scl = (shift)?(clk_100k):(sclTemp);
	/*---------counter---------*/
	always@(posedge clk or negedge rst_in_n)begin
		if(!rst_in_n)cnt_100k <= 9'd0;
		else begin
			if(cnt_100k < 9'd499)cnt_100k <= cnt_100k + 9'd1;
			else                 cnt_100k <= 9'd0;
		end
	end
	/*---------tick---------*/
	always@(posedge clk or negedge rst_in_n)begin
		if(!rst_in_n)tick_i2C <= 1'b0;
		else begin
			if(cnt_100k == 9'd498)tick_i2C <= 1'b1;
			else                  tick_i2C <= 1'b0;
		end
	end
	/*---------clk_100k---------*/
	always@(posedge clk or negedge rst_in_n)begin
		if(!rst_in_n)clk_100k <= 1'b0;
		else begin
			if(cnt_100k == 9'd100)     clk_100k <= 1'b1;
			else if(cnt_100k == 9'd400)clk_100k <= 1'b0;
			else                       clk_100k <= clk_100k;
		end
	end
	/*---------control en---------*/
	always@(posedge clk or negedge rst_in_n)begin
		if(!rst_in_n)en <= 1'b0;
		else begin
			if(sfr_addr == 8'h9A && sfr_data_out[0] && sfr_data_out[4])en <= 1'b1;
			else                                                       en <= 1'b0;
		end
	end
	/*---------control sfr_data_in---------*/
	always@(posedge clk or negedge rst_in_n)begin
		if(!rst_in_n)sfr_data_in <= 8'h00;
		else begin
			if(fstate == ACK && tick_i2C)sfr_data_in <= 8'h01;
			else if(sfr_addr == 8'h9B)   sfr_data_in <= 8'h00;
			else                         sfr_data_in <= sfr_data_in;
		end
	end
	/*---------catch & shift data---------*/
	always@(posedge clk or negedge rst_in_n)begin
		if(!rst_in_n)regData <= 8'h00;
		else begin
			if(sfr_addr == 8'h9C && sfr_wr)regData <= sfr_data_out;
			else begin
				if(tick_i2C)begin
					if(shift)regData <= {regData[6:0],1'b0};
					else     regData <= regData;
				end
				else regData <= regData;
			end
		end
	end
	/*---------fstate_state---------*/
	always@(posedge clk or negedge rst_in_n)begin
		if(!rst_in_n)fstate <= IDLE;
		else begin
			case(fstate)
				IDLE:begin
					if(en)fstate <= GO;
					else  fstate <= IDLE;
				end
				GO:begin
					if(tick_i2C)fstate <= START;
					else        fstate <= GO;
				end
				START:begin
					if(tick_i2C)fstate <= WAIT;
					else        fstate <= START;
				end
				WAIT:begin
					if(tick_i2C)fstate <= SHIFT;
					else        fstate <= WAIT;
				end
				SHIFT:begin
					if(index==3'd7&&tick_i2C)fstate <= ACK;
					else                     fstate <= SHIFT;
				end
				ACK:begin
					if(tick_i2C)begin
						if(times==2'd3)fstate <= STOP;
						else           fstate <= SHIFT;
					end 
					else              fstate <= ACK;
				end
				STOP:begin
					if(tick_i2C)fstate <= FINISH;
					else        fstate <= STOP;
				end
				FINISH:begin
					if(tick_i2C)fstate <= IDLE;
					else        fstate <= FINISH;
				end
				default:begin
					fstate <= IDLE;
				end
			endcase
		end
	end
	/*---------fstate_output---------*/
	always@(posedge clk or negedge rst_in_n)begin
		if(!rst_in_n)begin
			shift   <= 1'b0;
			index   <= 3'd0;
			times   <= 2'd0;
			sdaTemp <= 1'b1;
			sclTemp <= 1'd1;
		end
		else begin
			//if(tick_i2C)begin
			case(fstate)
				IDLE:begin
					shift   <= 1'b0;
					index   <= 3'd0;
					times   <= 2'd0;
					sdaTemp <= 1'b1;//HIGH
					sclTemp <= 1'd1;//HIGH
				end
				GO:begin
					shift   <= 1'b0;
					index   <= 3'd0;
					times   <= 2'd0;
					sdaTemp <= 1'b1;//HIGH
					sclTemp <= 1'd1;//HIGH
				end
				START:begin
					shift   <= 1'b0;
					index   <= 3'd0;
					times   <= 2'd0;
					sdaTemp <= 1'b0;//LOW
					sclTemp <= 1'd1;//HIGH
				end
				WAIT:begin
					shift   <= 1'b0;
					index   <= 3'd0;
					times   <= 2'd0;
					sdaTemp <= 1'b0;//LOW
					sclTemp <= 1'd1;//HIGH
				end
				SHIFT:begin
					shift   <= 1'b1;
					if(tick_i2C)begin
						if(index < 3'd7)index <= index + 3'd1;
						else            index <= 3'd0;
					end
					else index <= index;
					times   <= times;
					sdaTemp <= 1'b0;//X
					sclTemp <= 1'd0;//X
				end
				ACK:begin
					shift   <= 1'b1;
					index   <= 3'd0;
					if(tick_i2C)times <= times + 2'd1;
					else        times <= times;
					sdaTemp <= 1'b0;//X
					sclTemp <= 1'd0;//X
				end
				STOP:begin
					shift   <= 1'b0;
					index   <= 3'd0;
					times   <= 2'd0;
					sdaTemp <= 1'b0;//LOW
					sclTemp <= 1'd0;//LOW
				end
				FINISH:begin
					shift   <= 1'b0;
					index   <= 3'd0;
					times   <= 2'd0;
					sdaTemp <= 1'b0;//LOW
					sclTemp <= 1'd1;//HIGH
				end
				default:begin
					shift   <= 1'b0;
					index   <= 3'd0;
					times   <= 2'd0;
					sdaTemp <= 1'b1;//HIGH
					sclTemp <= 1'd1;//HIGH
				end
			endcase
		end
	end
	endmodule
```

## TestBench

```verilog
`timescale 1ns / 10ps
	/*
	`include "int_mem.v"
	`include "ext_mem.v"
	`include "rom_mem.v"
	`include "DW8051_core.v"
	`include "M24AA256.v"
	`include "sfr_mem.v"
	*/
	module tb_8051();
   parameter       clkx8 = 27.08;  // 48*64=3.072 MHZ,325/12=27.08

   reg clk; 
   reg por_n; 
   reg rst_in_n; 
   wire rst_out_n; 
   reg test_mode_n; 
   wire stop_mode_n; 
   wire idle_mode_n; 
   wire [7:0] sfr_addr; 
   wire [7:0] sfr_data_out; 
   wire [7:0] sfr_data_in; 
   wire sfr_wr; 
   wire sfr_rd; 
   wire [15:0] mem_addr; 
   wire [7:0] mem_data_out; 
   wire [7:0] mem_data_in; 
   wire mem_wr_r; 
   wire mem_rd_n; 
   wire mem_pswr_n;    
   
   wire mem_psrd_n; 
   wire mem_ale; 
   reg mem_ea_n; 
   wire int0_n; 
   wire int1_n; 
   wire int2; 
   wire int3_n; 
   wire int4; 
   wire int5_n; 
   wire pfi; 
   wire wdti; 
   wire rxd0_in; 
   wire rxd0_out, txd0; 
   wire rxd1_in; 
   wire rxd1_out, txd1; 
   wire t0; 
   wire t1; 
   wire t2; 
   wire t2ex; 
   wire t0_out, t1_out, t2_out; 
   wire port_pin_reg_n, p0_mem_reg_n, p0_addr_data_n, p2_mem_reg_n; 
   wire [7:0] iram_addr, iram_data_out, iram_data_in; 
   wire iram_rd_n, iram_we1_n, iram_we2_n; 
   wire [15:0] irom_addr; 
   wire [7:0] irom_data_out; 
   wire irom_rd_n, irom_cs_n; 
  
   wire [7:0] P0,P1,P2,P3;
   
   wire	i2c_sda;
   wire	i2c_scl;

   /*------------*/
	
	
	
   pullup(i2c_sda);
   pullup(i2c_scl);
   
   always
   begin
      #(clkx8/2) clk <= 1'b1 ;
      #(clkx8/2) clk <= 1'b0 ;
   end 

   initial
    begin
   
      //$fsdbDumpfile("tb_8051.fsdb");
      //$fsdbDumpfile("tb_8051_io.fsdb"); 
      //$fsdbDumpvars;
            
      mem_ea_n = 1'b1;              // 1: external ROM ,0: Internal ROM
      por_n = 1'b1 ;
      rst_in_n  = 1'b1;
      test_mode_n = 1'b1;
      #(clkx8*10);
      por_n = 1'b0 ;
      rst_in_n  = 1'b0;
      #(clkx8*10);
      por_n = 1'b1 ;
      rst_in_n  = 1'b1;
      
      wait((irom_addr == 16'h0B16) | (irom_addr == 16'h0B19));
      #1000;
      $stop;

    end
     
     
  
//--------------------------------------------------------------- 
// DW8051 instantiation: 
//--------------------------------------------------------------- 
DW8051_core u0 ( 
               .clk (clk), 
               .por_n (por_n), 
               .rst_in_n (rst_in_n), 
               .rst_out_n (rst_out_n), 
               .test_mode_n (test_mode_n), 
               .stop_mode_n (stop_mode_n), 
               .idle_mode_n (idle_mode_n), 
               .sfr_addr (sfr_addr), 
               .sfr_data_out (sfr_data_out), 
               .sfr_data_in (sfr_data_in), 
               .sfr_wr (sfr_wr), 
               .sfr_rd (sfr_rd), 
               .mem_addr (mem_addr), 
               .mem_data_out (mem_data_out), 
               .mem_data_in (mem_data_in), 
               .mem_wr_n (mem_wr_n), 
               .mem_rd_n (mem_rd_n), 
               .mem_pswr_n (mem_pswr_n), 
               .mem_psrd_n (mem_psrd_n), 
               .mem_ale (mem_ale), 
               .mem_ea_n (mem_ea_n),
               .int0_n (int0_n), 
               .int1_n (int1_n), 
               .int2 (int2), 
               .int3_n (int3_n), 
               .int4 (int4), 
               .int5_n (int5_n), 
               .pfi (pfi), 
               .wdti (wdti), 
               .rxd0_in (rxd0_in), 
               .rxd0_out (rxd0_out), 
               .txd0 (txd0), 
               .rxd1_in (rxd1_in), 
               .rxd1_out (rxd1_out), 
               .txd1 (txd1), 
               .t0 (t0), 
               .t1 (t1), 
               .t2 (t2), 
               .t2ex (t2ex), 
               .t0_out (t0_out), 
               .t1_out (t1_out), 
               .t2_out (t2_out), 
               .port_pin_reg_n (port_pin_reg_n), 
               .p0_mem_reg_n (p0_mem_reg_n), 
               .p0_addr_data_n (p0_addr_data_n), 
               .p2_mem_reg_n (p2_mem_reg_n), 
               .iram_addr (iram_addr), 
               .iram_data_out (iram_data_out), 
               .iram_data_in (iram_data_in), 
               .iram_rd_n (), 
               .iram_we1_n (iram_we1_n), 
               .iram_we2_n (iram_we2_n), 
               .irom_addr (irom_addr), 
               .irom_data_out (irom_data_out), 
               .irom_rd_n (irom_rd_n), 
               .irom_cs_n (irom_cs_n) 
               );
    /*                 
   sfr_mem      u1_sfr_mem(
                        .clk(clk),
                        .addr(sfr_addr),     
                        .data_in(sfr_data_out),
                        .data_out(sfr_data_in),
                        .wr_n(~sfr_wr),       
                        .rd_n(~sfr_rd)) ;  
        
    */               
   int_mem      u3_int_mem(
                        .clk(clk),
                        .addr(iram_addr),
                        .data_in(iram_data_in),
                        .data_out(iram_data_out),
                        .we1_n(iram_we1_n),
                        .we2_n(iram_we2_n),
                        .rd_n(iram_rd_n));  
                        
                                        
   ext_mem      u2_ext_mem(
                        .addr(mem_addr),
                        .data_in(mem_data_out),
                        .data_out(mem_data_in),
                        .wr_n(mem_wr_n),
                        .rd_n(mem_rd_n));    
                        
                          
  rom_mem       u4_rom_mem(
                        .addr(irom_addr),
                        .data_out(irom_data_out),                    
                        .rd_n(irom_rd_n),
                        .cs_n(irom_cs_n));  
//--------------------------------------------------------------- 
// EEPROM Model
//---------------------------------------------------------------                         
 M24AA256 u10(
    .A0(1'b0), 
    .A1(1'b0), 
    .A2(1'b0), 
    .WP(1'b0), 
    .SDA(i2c_sda), 
    .SCL(i2c_scl), 
    .RESET(~rst_in_n)
    );   
                                                         
hw3 u11(
    .rst_in_n(rst_in_n),
    .clk(clk),
    .sfr_wr(sfr_wr),
    .sfr_rd(sfr_rd),
    .sfr_addr(sfr_addr),
    .sfr_data_out(sfr_data_out),
    .sfr_data_in(sfr_data_in),
    .i2c_sda(i2c_sda),
    .i2c_scl(i2c_scl)
);                      
 
initial
  begin
  $monitor("time=%3d memory_000=0x%x memory_001=0x%x",$time,u10.MemoryByte_000[7:0],u10.MemoryByte_001[7:0]);
  end
   
endmodule
```

## Wave

![hw3%209f01092b49314c8986d43941b970e16b/Untitled.png](hw3%209f01092b49314c8986d43941b970e16b/Untitled.png)

![hw3%209f01092b49314c8986d43941b970e16b/Untitled%201.png](hw3%209f01092b49314c8986d43941b970e16b/Untitled%201.png)

![hw3%209f01092b49314c8986d43941b970e16b/Untitled%202.png](hw3%209f01092b49314c8986d43941b970e16b/Untitled%202.png)

## Monitor

![hw3%209f01092b49314c8986d43941b970e16b/Untitled%203.png](hw3%209f01092b49314c8986d43941b970e16b/Untitled%203.png)

## Compilation Report

![hw3_perfect%20cf6891640977461298a4e804fee3fa43/Untitled.png](hw3_perfect%20cf6891640977461298a4e804fee3fa43/Untitled.png)

## RTL Viewer

![hw3_perfect%20cf6891640977461298a4e804fee3fa43/Untitled%201.png](hw3_perfect%20cf6891640977461298a4e804fee3fa43/Untitled%201.png)

## State Machine

![hw3_perfect%20cf6891640977461298a4e804fee3fa43/Untitled%202.png](hw3_perfect%20cf6891640977461298a4e804fee3fa43/Untitled%202.png)