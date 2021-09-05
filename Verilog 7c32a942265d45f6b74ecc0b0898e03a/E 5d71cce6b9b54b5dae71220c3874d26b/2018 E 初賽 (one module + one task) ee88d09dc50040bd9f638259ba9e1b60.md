# 2018 E 初賽 (one module + one task)

[E_ICC2018_priliminary_univ_cell_based.pdf](2018%20E%20%E5%88%9D%E8%B3%BD%20(one%20module%20+%20one%20task)%20ee88d09dc50040bd9f638259ba9e1b60/E_ICC2018_priliminary_univ_cell_based.pdf)

```verilog
module LCD_CTRL(
    clk,
    reset,
    cmd,
    cmd_valid,
    IROM_Q,
    IROM_rd,
    IROM_A,
    IRAM_valid,
    IRAM_D,
    IRAM_A,
    busy,
    done
  );
  /*------------parameter------------*/
  localparam IDLE   = 3'd0;
  localparam GETCMD = 3'd1;
  localparam WAIT   = 3'd2;
  localparam PROCE  = 3'd3;
  localparam RAMOUT = 3'd4;
  localparam FINISH = 3'd5;
  /*------------portsdeclarations------------*/
  input      clk;
  input      reset;
  input [3:0]cmd;
  input      cmd_valid;
  input [7:0]IROM_Q;
  output     IROM_rd;
  output[5:0]IROM_A;
  output     IRAM_valid;
  output[7:0]IRAM_D;
  output[5:0]IRAM_A;
  output     busy;
  output     done;
  //outputreg
  reg        IROM_rd;
  reg   [5:0]IROM_A;
  reg        IRAM_valid;
  reg   [7:0]IRAM_D;
  reg   [5:0]IRAM_A;
  reg        busy;
  reg        done;
  /*------------variables------------*/
  reg   [7:0]dataReg[63:0];//storedata
  reg   [3:0]cmdReg;
  reg   [2:0]fstate;//stateregister
  reg   [6:0]j;//cleardataReg
  reg   [5:0]IRAM_AReg;
  reg   [7:0]pt0;
  reg   [7:0]pt1;
  reg   [7:0]pt2;
  reg   [7:0]pt3;
  wire  [7:0]dOutReg[3:0];
  wire       go;
  wire       ready;
  /*------------assignwire------------*/
  assign  go    = (&IROM_A);//gotonextstate
  assign  ready = (&IRAM_A);
  /*------------state------------*/
  always@(posedge clk or posedge reset)begin
    if(reset)fstate<=IDLE;
    else begin
      case(fstate)
        IDLE:begin
          if(go)fstate <= GETCMD;
          else  fstate <= IDLE;
        end
        GETCMD:begin
          fstate <= WAIT;
        end
        WAIT:begin
          fstate <= PROCE;
        end
        PROCE:begin
          if(cmdReg==4'd0)fstate <= RAMOUT;
          else            fstate <= GETCMD;
        end
        RAMOUT:begin
          if(ready)fstate <= FINISH;
          else     fstate <= RAMOUT;
        end
        FINISH:begin
          fstate <= IDLE;
        end
        default:fstate <= IDLE;
      endcase
    end
  end
  /*------------output------------*/
  always@(posedge clk or posedge reset)begin
    if(reset)begin
      IROM_rd    <= 1'b0;
      cmdReg     <= 4'd0;
      busy       <= 1'b1;
      IRAM_valid <= 1'b0;
      done       <= 1'b0;
      for(j=7'd0;j<7'd64;j=j+7'd1)begin
        dataReg[j] <= 8'd0;
      end
    end
    else begin
      case(fstate)
        IDLE:begin
          IROM_rd    <= 1'b1;
          busy       <= 1'b1;
          IRAM_valid <= 1'b0;
          done       <= 1'b0;
          dataReg[IROM_A] <= IROM_Q;
        end
        GETCMD:begin
          IROM_rd    <= 1'b0;
          busy       <= 1'b0;
          IRAM_valid <= 1'b0;
          done       <= 1'b0;
        end
        WAIT:begin
          if(cmd_valid)cmdReg <= cmd;
          else         cmdReg <= cmdReg;
          busy <= 1'b1;
          done <= 1'b0;
        end
        PROCE:begin
          IROM_rd    <= 1'b0;
          busy       <= 1'b1;
          IRAM_valid <= 1'b0;
          done       <= 1'b0;
			 imageProcess(cmdReg, dataReg[pt0],dataReg[pt1],dataReg[pt2],dataReg[pt3],dataReg[pt0],dataReg[pt1],dataReg[pt2],dataReg[pt3]);
        end
        RAMOUT:begin
          busy       <= 1'b1;
          IROM_rd    <= 1'b0;
          IRAM_valid <= 1'b1;
          done       <= 1'b0;
        end
        FINISH:begin
          IROM_rd    <= 1'b0;
          busy       <= 1'b1;
          IRAM_valid <= 1'b0;
          done       <= 1'b1;
        end
        default:begin
          IROM_rd    <= 1'b0;
          busy       <= 1'b1;
          IRAM_valid <= 1'b0;
          done       <= 1'b0;
        end
      endcase
    end
  end
  /*------------SenddatatoIRAM------------*/
  always@(posedge clk or posedge reset)begin
    if(reset)IRAM_D <= 8'd0;
    else     IRAM_D <= dataReg[IRAM_AReg];
  end
  /*------------countIROMAddress------------*/
  always@(posedge clk or posedge reset)begin
    if(reset)begin
      IROM_A <= 6'd0;
    end
    else begin
      if(IROM_rd)IROM_A <= IROM_A + 6'd1;
      else       IROM_A <= IROM_A;
    end
  end
  /*------------countIRAMAddress------------*/
  always@(posedge clk or posedge reset)begin
    if(reset)begin
      IRAM_A    <= 6'd0;
      IRAM_AReg <= 6'd0;
    end
    else begin
      if(IRAM_valid)IRAM_AReg <= IRAM_AReg + 6'd1;
      else          IRAM_AReg <= IRAM_AReg;
      IRAM_A<=IRAM_AReg;
    end
  end
  /*------------controleachcoordinates------------*/
  always@(posedge clk or posedge reset)begin
    if(reset)begin
      pt0<=8'h1b;
      pt1<=8'h1c;
      pt2<=8'h23;
      pt3<=8'h24;
    end
    else begin
      case(cmdReg)
        4'd1:begin//up
          if(pt0>8'h07&&fstate==PROCE)begin
            pt0<=pt0-8'h08;
            pt1<=pt1-8'h08;
            pt2<=pt0;
            pt3<=pt1;
          end
          else begin
            pt0 <= pt0;
            pt1 <= pt1;
            pt2 <= pt2;
            pt3 <= pt3;
          end
        end
        4'd2:begin//down
          if(pt3<8'h38&&fstate==PROCE)begin
            pt0 <= pt2;
            pt1 <= pt3;
            pt2 <= pt2 + 8'h08;
            pt3 <= pt3 + 8'h08;
          end
          else begin
            pt0 <= pt0;
            pt1 <= pt1;
            pt2 <= pt2;
            pt3 <= pt3;
          end
        end
        4'd3:begin//left
          if((pt2!=8'h08&&pt2!=8'h10&&pt2!=8'h18&&pt2!=8'h20&&pt2!=8'h28&&pt2!=8'h30&&pt2!=8'h38)&&(fstate==PROCE))begin
            pt0 <= pt0 - 8'h01;
            pt1 <= pt0;
            pt2 <= pt2 - 8'h01;
            pt3 <= pt2;
          end
          else begin
            pt0 <= pt0;
            pt1 <= pt1;
            pt2 <= pt2;
            pt3 <= pt3;
          end
        end
        4'd4:begin//right
          if((pt1!=8'h07&&pt1!=8'h0f&&pt1!=8'h17&&pt1!=8'h1f&&pt1!=8'h27&&pt1!=8'h2f&&pt1!=8'h37)&&(fstate==PROCE))begin
            pt0 <= pt1;
            pt1 <= pt1 + 8'h01;
            pt2 <= pt3;
            pt3 <= pt3 + 8'h01;
          end
          else begin
            pt0 <= pt0;
            pt1 <= pt1;
            pt2 <= pt2;
            pt3 <= pt3;
          end
        end
        default:begin
          pt0 <= pt0;
          pt1 <= pt1;
          pt2 <= pt2;
          pt3 <= pt3;
        end
      endcase
    end
  end
  /*------------task------------*/
  
  task imageProcess;
  input  [3:0]cmd;//command
  input  [7:0]pi0;//dataIn
  input  [7:0]pi1;//dataIn
  input  [7:0]pi2;//dataIn
  input  [7:0]pi3;//dataIn
  output [7:0]po0;//dataOut
  output [7:0]po1;//dataOut
  output [7:0]po2;//dataOut
  output [7:0]po3;//dataOut
  reg    [7:0]register;
  reg    [9:0]registerDiv;
  reg    [7:0]temp[1:0];
    casex(cmd)//output=input
      4'd0:begin
        po0 = pi0;
        po1 = pi1;
        po2 = pi2;
        po3 = pi3;
      end
      4'd5:begin//findMax
         temp[0] = (pi0>pi1)?(pi0):(pi1);
         temp[1] = (pi2>pi3)?(pi2):(pi3);
         register = (temp[0]>temp[1])?(temp[0]):(temp[1]);
         po0 = register;
         po1 = register;
         po2 = register;
         po3 = register;
      end
      4'd6:begin//findMin
         temp[0] = (pi0<pi1)?(pi0):(pi1);
         temp[1] = (pi2<pi3)?(pi2):(pi3);
         register = (temp[0]<temp[1])?(temp[0]):(temp[1]);
         po0 = register;
         po1 = register;
         po2 = register;
         po3 = register;
      end
      4'd7:begin//findAverage
         registerDiv = 10'd0;
         registerDiv = (pi0+pi1+pi2+pi3)>>2;
         po0 = registerDiv[7:0];
         po1 = registerDiv[7:0];
         po2 = registerDiv[7:0];
         po3 = registerDiv[7:0];
      end
      4'd8:begin//CounterclockwiseRotation
         po0 = pi1;
         po1 = pi3;
         po2 = pi0;
         po3 = pi2;
      end
      4'd9:begin//ClockwiseRotation
         po0 = pi2;
         po1 = pi0;
         po2 = pi3;
         po3 = pi1;
      end
      4'ha:begin//mirrorX
         po0 = pi2;
         po1 = pi3;
         po2 = pi0;
         po3 = pi1;
      end
      4'hb:begin//mirrorY
         po0 = pi1;
         po1 = pi0;
         po2 = pi3;
         po3 = pi2;
      end
      default:begin//4'dx
         po0 = pi0;
         po1 = pi1;
         po2 = pi2;
         po3 = pi3;
      end
    endcase
  endtask
  endmodule
```

## TestBench

```verilog
`timescale 1ns/10ps
`define CYCLE    19.3         	        // Modify your clock period here

`define tb3

`define SDFFILE  "./LCD_CTRL_syn.sdf"	// Modify your sdf file name

`ifdef tb1
  `define EXPECT "./tb1_goal.dat"
  `define CMD "./cmd1.dat"
  `define IMAGE "image1.dat"
`endif

`ifdef tb2
  `define EXPECT "./tb2_goal.dat"
  `define CMD "./cmd2.dat"
  `define IMAGE "image2.dat"
`endif

`ifdef tb3
  `define EXPECT "./tb3_goal.dat"
  `define CMD "./cmd3.dat"
  `define IMAGE "image3.dat"
`endif

module testfixture;
parameter IMAGE_N_PAT = 64;
parameter CMD_N_PAT = 46;
parameter t_reset = `CYCLE*2;
reg clk;
reg reset;
reg [6:0] err_IRAM;
reg [3:0] cmd;
reg cmd_valid;
reg [7:0]  out_mem[0:63];

wire IROM_rd;
wire [5:0] IROM_A;
wire IRAM_valid;
wire [7:0] IRAM_D;
wire [5:0] IRAM_A;
wire busy;
wire done;
wire [7:0]  IROM_Q;

integer i, j, k, l, err;

reg over;
reg   [3:0]   cmd_mem   [0:CMD_N_PAT-1];

	LCD_CTRL LCD_CTRL(.clk(clk), .reset(reset), 
		     .cmd(cmd), .cmd_valid(cmd_valid), 
                     .IROM_rd(IROM_rd), .IROM_A(IROM_A), .IROM_Q(IROM_Q), 
                     .IRAM_valid(IRAM_valid), .IRAM_D(IRAM_D), .IRAM_A(IRAM_A),
		     .busy(busy), .done(done));

	IROM  IROM_1(.IROM_rd(IROM_rd), .IROM_data(IROM_Q), .IROM_addr(IROM_A), .clk(clk), .reset(reset));

	IRAM IRAM_1 (.clk(clk), .IRAM_data(IRAM_D), .IRAM_addr(IRAM_A), .IRAM_valid(IRAM_valid));

//initial $sdf_annotate(`SDFFILE, top);

`ifdef SDF
	initial $sdf_annotate(`SDFFILE, LCD_CTRL);
`endif

initial	$readmemh (`CMD,    cmd_mem);
initial	$readmemh (`EXPECT, out_mem);
/*
initial begin

$fsdbDumpfile("LCD_CTRL.fsdb");
$fsdbDumpvars;
$fsdbDumpMDA;
end
*/

initial begin
   clk         = 1'b0;
   reset       = 1'b0;
   over	       = 1'b0;
   l	       = 0;
   err         = 0;   
end

always begin #(`CYCLE/2) clk = ~clk; end

initial begin
   @(negedge clk)  reset = 1'b1;
   #t_reset        reset = 1'b0;
                                  
end  

            
 always @(negedge clk)
begin

	begin
	if (l < CMD_N_PAT)
	begin
		if(!busy) 
		begin
        	cmd = cmd_mem[l];
        	cmd_valid = 1'b1;
		l=l+1;
		end  
		else
		cmd_valid = 1'b0;
	end
	else
	begin
		l=l;
		cmd_valid = 1'b0;
	end
	end
end

initial @(posedge done) 
begin
   for(k=0;k<64;k=k+1)begin
         if( IRAM_1.IRAM_M[k] !== out_mem[k]) 
		begin
         	$display("ERROR at %d:output %h !=expect %h ",k, IRAM_1.IRAM_M[k], out_mem[k]);
         	err = err+1 ;
		end
         else if ( out_mem[k] === 8'dx)
                begin
                $display("ERROR at %d:output %h !=expect %h ",k, IRAM_1.IRAM_M[k], out_mem[k]);
		err=err+1;
                end   
 over=1'b1;
end
        begin
	if (err === 0 &&  over===1'b1  )  begin
	            $display("All data have been generated successfully!\n");
	            $display("-------------------PASS-------------------\n");
		    #10 $finish;
	         end
	         else if( over===1'b1 )
		 begin 
	            $display("There are %d errors!\n", err);
	            $display("---------------------------------------------\n");
		    #10 $finish;
         	 end
	
	end
end

endmodule

//-----------------------------------------------------------------------
//-----------------------------------------------------------------------
module IROM (IROM_rd, IROM_data, IROM_addr, clk, reset);
input		IROM_rd;
input	[5:0] 	IROM_addr;
output	[7:0]	IROM_data;
input		clk, reset;

reg [7:0] sti_M [0:63];
integer i;

reg	[7:0]	IROM_data;

initial begin
	@ (negedge reset) $readmemb (`IMAGE , sti_M);
	end

always@(negedge clk) 
	if (IROM_rd) IROM_data <= sti_M[IROM_addr];
	
endmodule

//-----------------------------------------------------------------------
//-----------------------------------------------------------------------

module IRAM (IRAM_valid, IRAM_data, IRAM_addr, clk);
input		IRAM_valid;
input	[5:0] 	IRAM_addr;
input	[7:0]	IRAM_data;
input		clk;

reg [7:0] IRAM_M [0:63];
integer i;

initial begin
	for (i=0; i<=63; i=i+1) IRAM_M[i] = 0;
end

always@(negedge clk) 
	if (IRAM_valid) IRAM_M[ IRAM_addr ] <= IRAM_data;

endmodule
```

## Compilation Report

![2018%20E%20%E5%88%9D%E8%B3%BD%20(one%20module%20+%20one%20task)%20ee88d09dc50040bd9f638259ba9e1b60/Untitled.png](2018%20E%20%E5%88%9D%E8%B3%BD%20(one%20module%20+%20one%20task)%20ee88d09dc50040bd9f638259ba9e1b60/Untitled.png)

## RTL Simulation

## TB1

![2018%20E%20%E5%88%9D%E8%B3%BD%20(one%20module%20+%20one%20task)%20ee88d09dc50040bd9f638259ba9e1b60/Untitled%201.png](2018%20E%20%E5%88%9D%E8%B3%BD%20(one%20module%20+%20one%20task)%20ee88d09dc50040bd9f638259ba9e1b60/Untitled%201.png)

## TB2

![2018%20E%20%E5%88%9D%E8%B3%BD%20(one%20module%20+%20one%20task)%20ee88d09dc50040bd9f638259ba9e1b60/Untitled%202.png](2018%20E%20%E5%88%9D%E8%B3%BD%20(one%20module%20+%20one%20task)%20ee88d09dc50040bd9f638259ba9e1b60/Untitled%202.png)

## TB3

![2018%20E%20%E5%88%9D%E8%B3%BD%20(one%20module%20+%20one%20task)%20ee88d09dc50040bd9f638259ba9e1b60/Untitled%203.png](2018%20E%20%E5%88%9D%E8%B3%BD%20(one%20module%20+%20one%20task)%20ee88d09dc50040bd9f638259ba9e1b60/Untitled%203.png)

## Gate Level Simulation

## TB1

![2018%20E%20%E5%88%9D%E8%B3%BD%20(one%20module%20+%20one%20task)%20ee88d09dc50040bd9f638259ba9e1b60/Untitled%204.png](2018%20E%20%E5%88%9D%E8%B3%BD%20(one%20module%20+%20one%20task)%20ee88d09dc50040bd9f638259ba9e1b60/Untitled%204.png)

## TB2

![2018%20E%20%E5%88%9D%E8%B3%BD%20(one%20module%20+%20one%20task)%20ee88d09dc50040bd9f638259ba9e1b60/Untitled%205.png](2018%20E%20%E5%88%9D%E8%B3%BD%20(one%20module%20+%20one%20task)%20ee88d09dc50040bd9f638259ba9e1b60/Untitled%205.png)

## TB3

![2018%20E%20%E5%88%9D%E8%B3%BD%20(one%20module%20+%20one%20task)%20ee88d09dc50040bd9f638259ba9e1b60/Untitled%206.png](2018%20E%20%E5%88%9D%E8%B3%BD%20(one%20module%20+%20one%20task)%20ee88d09dc50040bd9f638259ba9e1b60/Untitled%206.png)

## Speed(Max)　 :　20.9

## Time : 5712725 ps