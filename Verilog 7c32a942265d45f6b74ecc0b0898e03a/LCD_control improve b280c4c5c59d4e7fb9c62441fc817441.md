# LCD_control improve

## LCD_control

```verilog
module LCD_control(
		iclkSys, 
		irst_n,
		ilcdrs,
		ilcdOn,		
		idataLCD,
		itick_start, 
		odataLCD,  
		olcdrs, 
		olcdrw, 
		olcdOn, 
		olcdEN,
		olcdReady
	);
	/*---------parameter Declarations---------*/
	//=======state=======//
	localparam IDLE    = 2'd0;
	localparam SENT    = 2'd1;
	localparam MAKEEN  = 2'd2;
	localparam FINISH  = 2'd3;
	//=======cnt=======//
	localparam cntStop = 12'd2499;
	localparam cntRise = 12'd624;
	localparam cntFall = 12'd1874;
	/*---------Ports Declarations---------*/
	input       iclkSys;
	input 	   irst_n; 
	input       ilcdrs;
	input       ilcdOn;
	input	 [7:0]idataLCD; 
	input	      itick_start; 
	output [7:0]odataLCD; 
	output	   olcdrs; 
	output	   olcdrw; 
	output	   olcdOn; 
	output	   olcdReady;
	output      olcdEN;
	reg    [7:0]odataLCD;
	reg	      olcdReady;
	reg         olcdEN;
	wire	      olcdrs; 
	wire	      olcdrw; 
	wire	      olcdOn;
	/*---------variables---------*/
	reg    [1:0]fstate;
	reg   [11:0]counter;
	reg         counterEN;
	reg         tick;
	reg         SHEN;
	reg         LDEN;
	/*---------assign wire---------*/
	assign olcdrs = ilcdrs;
	assign olcdOn = ilcdOn;
	assign olcdrw = 1'b0;
	/*---------counter---------*/
	always@(posedge iclkSys or negedge irst_n)begin
		if(!irst_n)counter <= 12'd0;
		else begin
			if(counterEN)begin
				if(counter < cntStop)counter <= counter + 12'd1;
				else                 counter <= 12'd0;
			end
			else counter <= 12'd0;
		end
	end
	/*---------tick---------*/
	always@(posedge iclkSys or negedge irst_n)begin
		if(!irst_n)tick <= 1'b0;
		else begin
			if(counter == cntStop-12'd1)tick <= 1'b1;
			else                        tick <= 1'b0;
		end
	end
	/*---------olcdEN---------*/
	always@(posedge iclkSys or negedge irst_n)begin
		if(!irst_n)olcdEN <= 1'b0;
		else begin
			if(SHEN)begin
				if(counter == cntRise)     olcdEN <= 1'b1;
				else if(counter == cntFall)olcdEN <= 1'b0;
				else                       olcdEN <= olcdEN;
			end
			else olcdEN <= 1'b0;
		end
	end
	/*---------fstate state---------*/
	always@(posedge iclkSys or negedge irst_n)begin
		if(!irst_n)fstate <= IDLE;
		else begin
			case(fstate)
				IDLE:begin
					if(itick_start)fstate <= SENT;
					else           fstate <= IDLE;
				end
				SENT:begin
					fstate <= MAKEEN;
				end
				MAKEEN:begin
					if(tick)fstate <= FINISH;
					else    fstate <= MAKEEN;
				end
				FINISH:begin
					if(tick)fstate <= IDLE;
					else    fstate <= FINISH;
				end
				default:fstate <= IDLE;
			endcase
		end
	end
	/*---------fstate output---------*/
	always@(posedge iclkSys or negedge irst_n)begin
		if(!irst_n)begin
			counterEN <= 1'b0;
			olcdReady <= 1'b0;
			odataLCD  <= 8'd0;
			SHEN      <= 1'b0;
		end
		else begin
			case(fstate)
				IDLE:begin
					counterEN <= 1'b0;
					olcdReady <= 1'b0;
					SHEN      <= 1'b0;
				end
				SENT:begin
					counterEN <= 1'b0;
					olcdReady <= 1'b0;
					odataLCD  <= idataLCD;
					SHEN      <= 1'b0;
				end
				MAKEEN:begin
					counterEN <= 1'b1;
					olcdReady <= 1'b0;
					SHEN      <= 1'b1;
				end
				FINISH:begin
					counterEN <= 1'b1;
					if(tick)olcdReady <= 1'b1;
					else    olcdReady <= 1'b0;
					SHEN      <= 1'b0;
				end
				default:begin
					counterEN <= 1'b0;
					olcdReady <= 1'b0;
					odataLCD  <= 8'd0;
					SHEN      <= 1'b0;
				end
			endcase
		end
	end
	endmodule
```

## TestBench

```verilog
`timescale 1ns / 1ns
module LCD_tb();
	reg sysClk, rst_n, startTick, rsSet, power;
	reg [7:0]lcd_di;
	wire endofpack, ready;
	wire lcd_en, lcd_rs, lcd_rw, lcd_on;
	wire [7:0]lcd_do;
//--------------------------------------------------------//
	localparam clkTime = 20/2;
	localparam setTime = 20;
	localparam durTime = 20*10000;
//--------------------------------------------------------//
/*
LCD UUT(	.sysClk(sysClk), 
			.rst_n(rst_n),
			.lcd_di(lcd_di),
			.startTick(startTick),
			.rsSet(rsSet),
			.power(power),
			.lcd_en(lcd_en),
			.lcd_rs(lcd_rs),
			.lcd_rw(lcd_rw),
			.lcd_do(lcd_do),
			.lcd_on(lcd_on),
			.endofpack(endofpack),
			.ready(ready));
			
			*/
	
	LCD_control UUT(
			.iclkSys(sysClk), 
			.irst_n(rst_n),
			.ilcdrs(rsSet),		
			.idataLCD(lcd_di), 
			.ilcdOn(power),
			.itick_start(startTick), 
			.odataLCD(lcd_do),  
			.olcdrs(lcd_rs), 
			.olcdrw(lcd_rw), 
			.olcdOn(lcd_on), 
			.olcdEN(lcd_en),
			.olcdReady(endofpack)
		);
		
//--------------------------------------------------------//		
	initial forever #clkTime sysClk = ~sysClk;
	initial begin
		sysClk = 0;
		rst_n  = 1;
		startTick = 0;
		lcd_di = 8'd0;
		rsSet  = 0;
		power  = 1;
		rst_n  = 0;
	#setTime rst_n = 1;
		lcd_di = 8'hA3;
		startTick = 1;	
	#setTime startTick = 0;
	#durTime;
	
		lcd_di = 8'h7C;
		startTick = 1;	
	#setTime startTick = 0;
		rsSet = 1;
	#durTime;
		rsSet = 0;
		
		lcd_di = 8'h81;
		startTick = 1;	
	#setTime startTick = 0;
	#durTime;
	
	#durTime	$stop;
	end
//--------------------------------------------------------//
endmodule
```

## Wave

![LCD_control%20improve%20b280c4c5c59d4e7fb9c62441fc817441/Untitled.png](LCD_control%20improve%20b280c4c5c59d4e7fb9c62441fc817441/Untitled.png)

## Compilation Report

![LCD_control%20improve%20b280c4c5c59d4e7fb9c62441fc817441/Untitled%201.png](LCD_control%20improve%20b280c4c5c59d4e7fb9c62441fc817441/Untitled%201.png)

## RTL Viewer

![LCD_control%20improve%20b280c4c5c59d4e7fb9c62441fc817441/Untitled%202.png](LCD_control%20improve%20b280c4c5c59d4e7fb9c62441fc817441/Untitled%202.png)

## State Machine

![LCD_control%20eb4469f82b8f416da406d7efb9dca2c1/Untitled%203.png](LCD_control%20eb4469f82b8f416da406d7efb9dca2c1/Untitled%203.png)