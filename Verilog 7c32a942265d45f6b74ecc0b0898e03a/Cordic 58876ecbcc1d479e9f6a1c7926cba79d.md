# Cordic

```verilog
module Cordic#
(
  parameter width = 7
)
(
  clkSys, 
  rst_n,
  en,
  x_in,
  y_in,
  radius,
  phase,
  error,
  ready
);
/*------parameter------*/
localparam IDLE    = 3'd0;
localparam LOAD    = 3'd1;
localparam FIRST   = 3'd2;
localparam SECOND  = 3'd3;
localparam THIRD   = 3'd4;
localparam FINISH  = 3'd5;
/*------ports declaration------*/
input                   clkSys;
input                   rst_n;
input                   en;
input  signed [width:0] x_in;
input  signed [width:0] y_in;
output        [width:0] radius;
output        [width:0] phase;
output        [width:0] error;
output                  ready;
reg                     ready;
reg    signed [width:0] radius;
reg    signed [width:0] phase;
reg    signed [width:0] error;
/*------variables------*/
reg    signed [width:0] x[ 3:0];
reg    signed [width:0] y[3:0];
reg    signed [width:0] z[3:0];
reg               [2:0] fstate;
/*------fstate_machine_state------*/
always@(posedge clkSys or negedge rst_n)begin
	if(!rst_n)fstate <= 2'd0;
	else begin
		case(fstate)
			IDLE:begin
				if(en)fstate <= LOAD;
				else  fstate <= IDLE;
			end
			LOAD:    fstate <= FIRST;
			FIRST:   fstate <= SECOND;
			SECOND:  fstate <= THIRD;
			THIRD:   fstate <= FINISH;
			FINISH:  fstate <= IDLE;
			default :fstate <= IDLE;
		endcase
	end
end
/*------fstate_machine_output------*/
always@(posedge clkSys or negedge rst_n)begin
	if(!rst_n)begin
		x[0]<=0;x[1]<=0;x[2]<=0;x[3]<=0;
		y[0]<=0;y[1]<=0;y[2]<=0;y[3]<=0;
		z[0]<=0;z[1]<=0;z[2]<=0;z[3]<=0;
		ready  <= 1'b0;
		radius <= 0;
		phase  <= 0;
		error  <= 0;
	end
	else begin
		case(fstate)
			IDLE:begin
				x[0]<=0;x[1]<=0;x[2]<=0;x[3]<=0;
				y[0]<=0;y[1]<=0;y[2]<=0;y[3]<=0;
				z[0]<=0;z[1]<=0;z[2]<=0;z[3]<=0;
				ready <= 1'b0;
			end
			LOAD:begin
				if(x_in >= 0)begin
					x[0] <= x_in;
					y[0] <= y_in;
					z[0] <= 8'd0;
				end
				else if(y_in >= 0)begin
					x[0] <= y_in;
					y[0] <= -x_in;
					z[0] <= 8'd90;
				end
				else begin
					x[0] <= -y_in;
					y[0] <= x_in;
					z[0] <= -8'd90;
				end
			end
			FIRST:begin
				if(y[0] >= 0)begin
					x[1] <= x[0] + y[0];
					y[1] <= y[0] - x[0];
					z[1] <= z[0] + 8'd45;
				end
				else begin
					x[1] <= x[0] - y[0];
					y[1] <= y[0] + x[0];
					z[1] <= z[0] - 8'd45;
				end
			end
			SECOND:begin
				if(y[1] >= 0)begin
					x[2] <= x[1] + ({y[1][width], y[1][width:1]});//y[1]>>>1
					y[2] <= y[1] - ({x[1][width], x[1][width:1]});//x[1]>>>1
					z[2] <= z[1] + 8'd26;
				end
				else begin
					x[2] <= x[1] - ({y[1][width], y[1][width:1]});
					y[2] <= y[1] + ({x[1][width], x[1][width:1]});
					z[2] <= z[1] - 8'd26;
				end
			end
			THIRD:begin
				if(y[2] >= 0)begin
					x[3] <= x[2] + ({{2{y[2][width]}}, y[2][width:2]});//y[2]>>>2
					y[3] <= y[2] - ({{2{x[2][width]}}, x[2][width:2]});//x[2]>>>2
					z[3] <= z[2] + 8'd14;
				end
				else begin
					x[3] <= x[2] - ({{2{y[2][width]}}, y[2][width:2]});
					y[3] <= y[2] + ({{2{x[2][width]}}, x[2][width:2]});
					z[3] <= z[2] - 8'd14;
				end
			end
			FINISH:begin
				radius <= x[3];
				phase  <= z[3];
				error  <= y[3];
				ready  <= 1'b1;
			end
			default:begin
				x[0]<=0;x[1]<=0;x[2]<=0;x[3]<=0;
				y[0]<=0;y[1]<=0;y[2]<=0;y[3]<=0;
				z[0]<=0;z[1]<=0;z[2]<=0;z[3]<=0;
				ready  <= 1'b0;
				radius <= 0;
				phase  <= 0;
				error  <= 0;
			end
		endcase
	end
end
endmodule
```

## TestBench

```verilog
`timescale 1ns/1ns
	module tb_Cordic();
	localparam width = 7;
	reg                    clkSys;
	reg                    rst_n;
	reg                    en;
	reg   signed [width:0] x_in;
	reg   signed [width:0] y_in;
	wire  signed [width:0] radius;
	wire  signed [width:0] phase;
	wire  signed [width:0] error;
	wire                   ready;
	Cordic UUT
	(
		.clkSys(clkSys), 
		.rst_n(rst_n),
		.en(en),
	   .x_in(x_in),
      .y_in(y_in),
		.radius(radius),
		.phase(phase),
		.error(error),
		.ready(ready)
	);
	always #5 clkSys = ~clkSys;
	initial begin
		clkSys = 0;
		rst_n = 0;
		x_in = 0;
		y_in = 0;
		en = 0;
		repeat(5)@(posedge clkSys)rst_n = 0;
		rst_n = 1;
		test(-41,55);
		test(30, 40);
		test(20, 21);
		test(7,  24);
		test(64, 32);
		test(-35,15);
		test(-15,35);
		#1000 $stop;
	end
	/*----------task---------*/
	task test;
	input signed [width:0] in1;
	input signed [width:0] in2;
	  begin
	    #30;
	    x_in = in1;
		  y_in = in2;
       #10 en = 1;
       #10 en = 0;
		 wait (ready == 1);
     end
	endtask
	
	endmodule
```

![Cordic%2058876ecbcc1d479e9f6a1c7926cba79d/Untitled.png](Cordic%2058876ecbcc1d479e9f6a1c7926cba79d/Untitled.png)

## Compil

![Cordic%2058876ecbcc1d479e9f6a1c7926cba79d/Untitled%201.png](Cordic%2058876ecbcc1d479e9f6a1c7926cba79d/Untitled%201.png)

## RTL Viewer

![Cordic%2058876ecbcc1d479e9f6a1c7926cba79d/Untitled%202.png](Cordic%2058876ecbcc1d479e9f6a1c7926cba79d/Untitled%202.png)

## Fmax

![Cordic%2058876ecbcc1d479e9f6a1c7926cba79d/Untitled%203.png](Cordic%2058876ecbcc1d479e9f6a1c7926cba79d/Untitled%203.png)