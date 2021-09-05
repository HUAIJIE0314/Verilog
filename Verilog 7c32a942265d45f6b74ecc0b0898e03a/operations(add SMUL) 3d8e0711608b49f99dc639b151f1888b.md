# operations(add SMUL)

```verilog
module operations(
  clkSys,//input  clkSys
  rst,   //input  rst
  en,    //input  en
  a0,    //input  a0
  a1,    //input  a1
  a2,    //input  a2
  a3,    //input  a3
  b0,    //input  b0
  b1,    //input  b1
  b2,    //input  b2
  b3,    //input  b3
  mode,  //input  mode
  dout0, //output dout0
  dout1, //output dout1
  dout2, //output dout2
  dout3, //output dout3
  ready  //output ready
);
/*---------state parameter---------*/
localparam IDLE    = 5'd0; //IDLE
localparam START   = 5'd1; //START
localparam NOT     = 5'd2; //Logical operation NOT
localparam AND     = 5'd3; //Logical operation AND
localparam OR      = 5'd4; //Logical operation OR
localparam XOR     = 5'd5; //Logical operation XOR
localparam ADDIN   = 5'd6; //Arithmetic operation Addition (In-Place)
localparam ADDOUT  = 5'd7; //Arithmetic operation Addition (Out-Of-Place)
localparam SUBIN   = 5'd8; //Arithmetic operation Subtraction (In-Place)
localparam SUBOUT  = 5'd9; //Arithmetic operation Subtraction (Out-Of-Place)
localparam MUL     = 5'd10;//Arithmetic operation Multiplication
localparam SMUL    = 5'd11;//Arithmetic operation Signed Multiplication
localparam TWOCOMP = 5'd12;//operation 2's Complement
localparam ABS     = 5'd13;//operation Absolute
localparam STORE   = 5'd30;//STORE data from tempR
localparam FINISH  = 5'd31;//FINISH
/*---------ports declarations---------*/
input        clkSys;//clock
input        rst;   //reset(active high asynchronous)
input        en;    //enable state machine
input  [3:0] a0;    //A[0]
input  [3:0] a1;    //A[1]
input  [3:0] a2;    //A[2]
input  [3:0] a3;    //A[3]
input  [3:0] b0;    //B[0]
input  [3:0] b1;    //B[1]
input  [3:0] b2;    //B[2]
input  [3:0] b3;    //B[3]
input  [3:0] mode;  //mode
output [3:0] dout0; //R[0]
output [3:0] dout1; //R[1]
output [3:0] dout2; //R[2]
output [3:0] dout3; //R[3]
output       ready; //done for state machine
reg    [3:0] dout0; //reg
reg    [3:0] dout1; //reg
reg    [3:0] dout2; //reg
reg    [3:0] dout3; //reg
reg          ready; //reg
/*---------variables---------*/
reg    [4:0] fstate;      //state register
reg    [3:0] tempA [3:0]; //temp A 
reg    [3:0] tempB [3:0]; //temp B 
reg    [3:0] tempR [3:0]; //temp R 
reg    [1:0] column;      //column of matrix     
reg    [3:0] signReg;     //signed bit for signed multiplication
reg    [3:0] Cr;          //Carry bit for Addition operation
reg    [2:0] j;           //order of operations(1st, 2nd, 3rd)(from LUTs)
wire   [1:0] index1 [3:0];//index of multiplication
wire   [1:0] index2 [3:0];//index of multiplication
wire   [1:0] index3 [3:0];//index of multiplication
reg          colInc;      //the flag of increasing column
reg          flagSMUL;    //flag
reg          flagSTORE;   //flag
reg          flagCOMP;    //flag
integer      i;           //use to initialize the matrix and index for each data in A,B,R
/*---------assign wire---------*/
assign {index1[0],index1[1], index1[2], index1[3]} = {2'd0, 2'd1, 2'd0, 2'd1};
assign {index2[0],index2[1], index2[2], index2[3]} = {2'd0, 2'd0, 2'd1, 2'd1};
assign {index3[0],index3[1], index3[2], index3[3]} = {2'd0, 2'd1, 2'd1, 2'd2};
//================= function table ================
// ----------------------------------------------- 
//|           operation          |      mode      |
//| ----------------------------------------------|
//|   NOT                        |      4'd0      |
//|   AND                        |      4'd1      |
//|   OR                         |      4'd2      |
//|   XOR                        |      4'd3      |
//|   Addition(In-Place)         |      4'd4      |
//|   Addition(Out-Of-Place)     |      4'd5      |
//|   Subtraction(In-Place)      |      4'd6      |
//|   Subtraction(Out-Of-Place)  |      4'd7      |
//|   Multiplication             |      4'd8      |
//|   Signed Multiplication      |      4'd9      |
//|   2's complement             |      4'd10     |
//|   Absolute                   |      4'd11     |
// ----------------------------------------------- 
//=================================================
/*---------state machine(state)---------*/
always@(posedge clkSys or posedge rst)begin
  if(rst)fstate <= IDLE;
  else begin
    case(fstate)
      IDLE:begin 
        if(en)fstate <= START;
        else  fstate <= IDLE;
      end
      START:begin
        case(mode)//mode select
          4'd0:   fstate <= NOT;
          4'd1:   fstate <= AND;
          4'd2:   fstate <= OR;
          4'd3:   fstate <= XOR;
          4'd4:   fstate <= ADDIN;
          4'd5:   fstate <= ADDOUT;
          4'd6:   fstate <= SUBIN;
          4'd7:   fstate <= SUBOUT;
          4'd8:   fstate <= MUL;
          4'd9:   fstate <= SMUL;
          4'd10:  fstate <= TWOCOMP;
          4'd11:  fstate <= ABS;
          default:fstate <= IDLE;
        endcase
      end
      NOT:begin
        if(column == 2'd3)fstate <= FINISH;
        else              fstate <= NOT;
      end
      AND:begin
        if(column == 2'd3)fstate <= FINISH;
        else              fstate <= AND;
      end
      OR:begin
        if(column == 2'd3)fstate <= FINISH;
        else              fstate <= OR;
      end
      XOR:begin
        if(column == 2'd3)fstate <= FINISH;
        else              fstate <= XOR;
      end
      ADDIN:begin
        if(column == 2'd3)fstate <= FINISH;
        else              fstate <= ADDIN;
      end
      ADDOUT:begin
        if(column == 2'd3)fstate <= FINISH;
        else              fstate <= ADDOUT;
      end
      SUBIN:begin
        if(column == 2'd3)fstate <= FINISH;
        else              fstate <= SUBIN;
      end
      SUBOUT:begin
        if(column == 2'd3)fstate <= FINISH;
        else              fstate <= SUBOUT;
      end
      MUL:begin
        if(column == 2'd3)fstate <= FINISH;
        else              fstate <= MUL;
      end
      SMUL:begin
        if(column == 2'd3)fstate <= STORE;
        else if(flagSMUL) fstate <= SMUL;
        else              fstate <= ABS;
      end
      TWOCOMP:begin
        if(column == 2'd3)fstate <= FINISH;
        else              fstate <= TWOCOMP;
      end
      ABS:begin
        if(column == 2'd2)            fstate <= FINISH;
        else if(flagSMUL && !flagCOMP)fstate <= STORE;
        else                          fstate <= ABS;
      end
      STORE:begin
        if(!flagSTORE || flagCOMP)fstate <= ABS;
        else                      fstate <= SMUL;
      end
      FINISH: fstate <= IDLE;
      default:fstate <= IDLE;
    endcase 
  end
end

/*---------state machine(output)---------*/
always @(posedge clkSys or posedge rst) begin
  if(rst)begin
    for(i=0;i<4;i=i+1)begin
      tempA[i] <= 4'd0;
      tempB[i] <= 4'd0;
      tempR[i] <= 4'd0;
    end
    ready     <= 1'b0;
    colInc    <= 1'b0;
    Cr        <= 4'd0;
    signReg   <= 4'd0;
    flagSMUL  <= 1'b0;
    flagSTORE <= 1'b0;
    flagCOMP  <= 1'b0;
    {dout3,dout2,dout1,dout0} <= 16'd0;
  end
  else begin
    case(fstate)
      IDLE:begin 
        ready     <= 1'b0;
        colInc    <= 1'b0;
        Cr        <= 4'd0;
        flagSMUL  <= 1'b0;
        flagSTORE <= 1'b0;
        flagCOMP  <= 1'b0;
        for(i=0;i<4;i=i+1)begin    
          tempR[i] <= 4'd0;
        end
      end
      START:begin
        {tempA[3],tempA[2],tempA[1],tempA[0]} <= {a3,a2,a1,a0};
        {tempB[3],tempB[2],tempB[1],tempB[0]} <= {b3,b2,b1,b0};
        signReg <= {a3[1]^b3[1],a2[1]^b2[1],a1[1]^b1[1],a0[1]^b0[1]};
        if(mode == 4'd9)colInc  <= 1'b0;
        else            colInc  <= 1'b1;
      end
      NOT:begin
        for(i=0;i<4;i=i+1)begin
          opNOT(tempA[i][column], tempR[i][column]);
        end
      end
      AND:begin
        for(i=0;i<4;i=i+1)begin
          opAND(tempA[i][column], tempB[i][column], tempR[i][column]);
        end
      end
      OR:begin
        for(j=3'd0;j<3'd3;j=j+3'd1)begin//1st, 2nd, 3rd
          for(i=0;i<4;i=i+1)begin
            opOR(j, tempA[i][column], tempB[i][column], tempR[i][column], tempR[i][column]);
          end 
        end
      end
      XOR:begin
        for(j=3'd0;j<3'd2;j=j+3'd1)begin//1st, 2nd
          for(i=0;i<4;i=i+1)begin
            opXOR(j, tempA[i][column], tempB[i][column], tempR[i][column], tempR[i][column]);
          end 
        end
      end
      ADDIN:begin
        for(j=3'd0;j<3'd4;j=j+3'd1)begin//1st, 2nd, 3rd, 4th
          for(i=0;i<4;i=i+1)begin
            opADDIN(j, Cr[i], tempA[i][column], tempB[i][column], Cr[i], tempB[i][column]);
          end 
        end
      end
      ADDOUT:begin
        for(j=3'd0;j<3'd5;j=j+3'd1)begin//1st, 2nd, 3rd, 4th, 5th
          for(i=0;i<4;i=i+1)begin
            opADDOUT(j, Cr[i], tempA[i][column], tempB[i][column], tempR[i][column], Cr[i], tempR[i][column]);
          end 
        end
      end
      SUBIN:begin
        for(j=3'd0;j<3'd4;j=j+3'd1)begin//1st, 2nd, 3rd, 4th
          for(i=0;i<4;i=i+1)begin
            opSUBIN(j, Cr[i], tempB[i][column], tempA[i][column], Cr[i], tempA[i][column]);//!!!!LUTs is B = B - A --> A = A - B
          end 
        end
      end
      SUBOUT:begin
        for(j=3'd0;j<3'd5;j=j+3'd1)begin//1st, 2nd, 3rd, 4th, 5th
          for(i=0;i<4;i=i+1)begin
            opSUBOUT(j, Cr[i], tempB[i][column], tempA[i][column], tempR[i][column], Cr[i], tempR[i][column]);//!!!!LUTs is R = B - A --> R = A - B
          end 
        end
      end
      MUL:begin//unsigned multiplication
        for(j=3'd0;j<3'd4;j=j+3'd1)begin//1st, 2nd, 3rd, 4th
          for(i=0;i<4;i=i+1)begin
            opMUL(j, tempR[i][index2[column]+2'd2], tempA[i][index1[column]], tempB[i][index2[column]], tempR[i][index3[column]], tempR[i][index2[column]+2'd2], tempR[i][index3[column]]);   
          end
        end
      end
      SMUL:begin//signed multiplication
        if(!flagSMUL)flagSMUL <= 1'b1;
        else begin
          flagCOMP <= 1'b1;
          for(j=3'd0;j<3'd4;j=j+3'd1)begin//1st, 2nd, 3rd, 4th
            for(i=0;i<4;i=i+1)begin
              opMUL(j, tempR[i][index2[column]+2'd2], tempA[i][index1[column]], tempB[i][index2[column]], tempR[i][index3[column]], tempR[i][index2[column]+2'd2], tempR[i][index3[column]]);   
            end
          end
          if(column == 2'd3 && flagCOMP)colInc <= 1'b0;
          else                          colInc <= 1'b1;
        end
      end
      TWOCOMP:begin
        for(j=3'd0;j<3'd3;j=j+3'd1)begin//1st, 2nd, 3rd
          for(i=0;i<4;i=i+1)begin
            opTWOCOMP(j, Cr[i], tempA[i][column], tempR[i][column], Cr[i], tempR[i][column]);
          end 
        end
      end
      ABS:begin
        if(!flagSMUL || !flagSTORE)begin//convert A for normal ABS function
          for(j=3'd0;j<3'd4;j=j+3'd1)begin//1st, 2nd, 3rd, 4th
            for(i=0;i<4;i=i+1)begin
              opABS(j, Cr[i], tempA[i][3], tempA[i][column], tempR[i][column], Cr[i], tempR[i][column]);
            end 
          end
        end
        else if(!flagSTORE)begin//convert A for sign multiplication
          for(j=3'd0;j<3'd4;j=j+3'd1)begin//1st, 2nd, 3rd, 4th
            for(i=0;i<4;i=i+1)begin
              opABS(j, Cr[i], tempA[i][1], tempA[i][column], tempR[i][column], Cr[i], tempR[i][column]);
            end 
          end
        end
        else if(flagCOMP)begin//convert the result of signed multiplication
          for(j=3'd0;j<3'd4;j=j+3'd1)begin//1st, 2nd, 3rd, 4th
            for(i=0;i<4;i=i+1)begin
              opABS(j, Cr[i], signReg[i], tempA[i][column], tempR[i][column], Cr[i], tempR[i][column]);
            end 
          end
        end
        else begin//convert B
          for(j=3'd0;j<3'd4;j=j+3'd1)begin//1st, 2nd, 3rd, 4th
            for(i=0;i<4;i=i+1)begin
              opABS(j, Cr[i], tempB[i][1], tempB[i][column], tempR[i][column], Cr[i], tempR[i][column]);
            end 
          end
        end
      end
      STORE:begin
        flagSTORE <= 1'b1;
        Cr        <= 4'd0;//initial the Cr
        if(flagSTORE)colInc <= 1'b1;
        else         colInc <= colInc;
        if(!flagSTORE || flagCOMP){tempA[3],tempA[2],tempA[1],tempA[0]} <= {tempR[3],tempR[2],tempR[1],tempR[0]};
        else                      {tempB[3],tempB[2],tempB[1],tempB[0]} <= {tempR[3],tempR[2],tempR[1],tempR[0]};
        //initial the temp
        for(i=0;i<4;i=i+1)begin    
          tempR[i] <= 4'd0;
        end
      end
      FINISH:begin
        ready   <= 1'b1;
        colInc  <= 1'b0;
        if(!flagSMUL)begin
          dout3 <= tempR[3];
          dout2 <= tempR[2];
          dout1 <= tempR[1];
          dout0 <= tempR[0];
        end 
        else begin
          dout3 <= {signReg[3]&(|tempR[3][2:0]),tempR[3][2:0]};
          dout2 <= {signReg[2]&(|tempR[2][2:0]),tempR[2][2:0]};
          dout1 <= {signReg[1]&(|tempR[1][2:0]),tempR[1][2:0]};
          dout0 <= {signReg[0]&(|tempR[0][2:0]),tempR[0][2:0]};
        end
      end
      default:begin
        for(i=0;i<4;i=i+1)begin
          tempA[i] <= 4'd0;
          tempB[i] <= 4'd0;
          tempR[i] <= 4'd0;
        end
        ready     <= 1'b0;
        colInc    <= 1'b0;
        Cr        <= 4'd0;
        flagSMUL  <= 1'b0;
        flagSTORE <= 1'b0;
        flagCOMP  <= 1'b0;
      end
    endcase 
  end
end

/*---------column---------*/
always@(posedge clkSys or posedge rst)begin
  if(rst)         column <= 2'd0;
  else if(colInc) column <= column + 2'd1;    
  else            column <= 2'd0;
end

/*------------------task of each function------------------*/
/*---------NOT---------*/
task opNOT;
  input  A;
  output R;
  begin
    R = ~A;
  end
endtask
/*---------AND---------*/
task opAND;
  input  A;
  input  B;
  output R;
  begin
    R = A & B;
  end
endtask
/*---------OR---------*/
task opOR;
  input [2:0] order;
  input       A;
  input       B;
  input       Rin;
  output      Rout;
  begin
    case(order)
      3'd0:begin
        if(!B && A)Rout = 1;
        else       Rout = Rin;
      end
      3'd1:begin
        if(B && !A)Rout = 1;
        else       Rout = Rin;
      end
      3'd2:begin
        if (B && A)Rout = 1;
        else       Rout = Rin;
      end
      default:Rout = Rin;
    endcase
  end
endtask
/*---------XOR---------*/
task opXOR;
  input [2:0] order;
  input       A;
  input       B;
  input       Rin;
  output      Rout;
  begin
    case(order)
      3'd0:begin
        if(!B && A)Rout = 1;
        else       Rout = Rin;
      end
      3'd1:begin
        if(B && !A)Rout = 1;
        else       Rout = Rin;
      end
      default:Rout = Rin;
    endcase
  end
endtask
/*---------ADDIN---------*/
task opADDIN;
  input [2:0] order;
  input       Crin;
  input       A;
  input       B;
  output      Crout;
  output      Bout;
  begin
    case(order)
      3'd0:begin
        if(!Crin && B && A) {Crout, Bout} = {1'b1, 1'b0};
        else                {Crout, Bout} = {Crin, B};
      end
      3'd1:begin
        if(!Crin && !B && A){Crout, Bout} = {1'b0, 1'b1};
        else                {Crout, Bout} = {Crin, B};
      end
      3'd2:begin
        if(Crin && !B && !A){Crout, Bout} = {1'b0, 1'b1};
        else                {Crout, Bout} = {Crin, B};
      end
      3'd3:begin
        if(Crin && B && !A) {Crout, Bout} = {1'b1, 1'b0};
        else                {Crout, Bout} = {Crin, B};
      end
      default:{Crout, Bout} = {Crin, B};
    endcase
  end
endtask
/*---------ADDOUT---------*/
task opADDOUT;
  input [2:0] order;
  input       Crin;
  input       A;
  input       B;
  input       Rin;
  output      Crout;
  output      Rout;
  begin
    case(order)
      3'd0:begin
        if(!Crin && !B && A){Crout, Rout} = {1'b0, 1'b1};
        else                {Crout, Rout} = {Crin, Rin};
      end
      3'd1:begin
        if(!Crin && B && !A){Crout, Rout} = {1'b0, 1'b1};
        else                {Crout, Rout} = {Crin, Rin};
      end
      3'd2:begin
        if(Crin && !B && !A){Crout, Rout} = {1'b0, 1'b1};
        else                {Crout, Rout} = {Crin, Rin};
      end
      3'd3:begin
        if(Crin && B && A)  {Crout, Rout} = {1'b1, 1'b1};
        else                {Crout, Rout} = {Crin, Rin};
      end
      3'd4:begin
        if(!Crin && B && A) {Crout, Rout} = {1'b1, 1'b0};
        else                {Crout, Rout} = {Crin, Rin};
      end
      default:{Crout, Rout} = {Crin, Rin};
    endcase
  end
endtask
/*---------SUBIN---------*/
task opSUBIN;
  input [2:0] order;
  input       Crin;
  input       A;
  input       B;
  output      Crout;
  output      Bout;
  begin
    case(order)
      3'd0:begin
        if(!Crin && !B && A){Crout, Bout} = {1'b1, 1'b1};
        else                {Crout, Bout} = {Crin, B};
      end
      3'd1:begin
        if(!Crin && B && A) {Crout, Bout} = {1'b0, 1'b0};
        else                {Crout, Bout} = {Crin, B};
      end
      3'd2:begin
        if(Crin && B && !A) {Crout, Bout} = {1'b0, 1'b0};
        else                {Crout, Bout} = {Crin, B};
      end
      3'd3:begin
        if(Crin && !B && !A){Crout, Bout} = {1'b1, 1'b1};
        else                {Crout, Bout} = {Crin, B};
      end
      default:{Crout, Bout} = {Crin, B};
    endcase
  end
endtask
/*---------SUBOUT---------*/
task opSUBOUT;
  input [2:0] order;
  input       Crin;
  input       A;
  input       B;
  input       Rin;
  output      Crout;
  output      Rout;
  begin
    case(order)
      3'd0:begin
        if(!Crin && !B && A){Crout, Rout} = {1'b1, 1'b1};
        else                {Crout, Rout} = {Crin, Rin};
      end
      3'd1:begin
        if(!Crin && B && !A){Crout, Rout} = {1'b0, 1'b1};
        else                {Crout, Rout} = {Crin, Rin};
      end
      3'd2:begin
        if(Crin && !B && !A){Crout, Rout} = {1'b1, 1'b1};
        else                {Crout, Rout} = {Crin, Rin};
      end
      3'd3:begin
        if(Crin && B && !A) {Crout, Rout} = {1'b0, 1'b0};
        else                {Crout, Rout} = {Crin, Rin};
      end
      3'd4:begin
        if(Crin && B && A)  {Crout, Rout} = {1'b1, 1'b1};
        else                {Crout, Rout} = {Crin, Rin};
      end
      default:{Crout, Rout} = {Crin, Rin};
    endcase
  end
endtask
/*---------MUL---------*/
task opMUL;
  input [2:0] order;
  input       Crin;
  input       A;
  input       B;
  input       Rin;
  output      Crout;
  output      Rout;
  begin
    case(order)
      3'd0:begin
        if(!Crin && Rin && B && A) {Crout, Rout} = {1'b1, 1'b0};
        else                       {Crout, Rout} = {Crin, Rin};
      end
      3'd1:begin
        if(!Crin && !Rin && B && A){Crout, Rout} = {1'b0, 1'b1};
        else                       {Crout, Rout} = {Crin, Rin};
      end
      3'd2:begin
        if(Crin && !Rin && !B && A){Crout, Rout} = {1'b0, 1'b1};
        else                       {Crout, Rout} = {Crin, Rin};
      end
      3'd3:begin
        if(Crin && Rin && !B && A) {Crout, Rout} = {1'b1, 1'b0};
        else                       {Crout, Rout} = {Crin, Rin};
      end
      default:{Crout, Rout} = {Crin, Rin};
    endcase
  end
endtask
/*---------TWOCOMP---------*/
task opTWOCOMP;
  input [2:0] order;
  input       Crin;
  input       A;
  input       Rin;
  output      Crout;
  output      Rout;
  begin
    case(order)
      3'd0:begin
        if(Crin && !A){Crout, Rout} = {1'b1, 1'b1};
        else          {Crout, Rout} = {Crin, Rin};
      end
      3'd1:begin
        if(Crin && A) {Crout, Rout} = {1'b1, 1'b0};
        else          {Crout, Rout} = {Crin, Rin};
      end
      3'd2:begin
        if(!Crin && A){Crout, Rout} = {1'b1, 1'b1};
        else          {Crout, Rout} = {Crin, Rin};
      end
      default:{Crout, Rout} = {Crin, Rin};
    endcase
  end
endtask
/*---------ABS---------*/
task opABS;
  input [2:0] order;
  input       Crin;
  input       sign;
  input       A;
  input       Rin;
  output      Crout;
  output      Rout;
  begin
    case(order)
      3'd0:begin
        if(!Crin && !sign && A){Crout, Rout} = {1'b0, 1'b1};
        else                   {Crout, Rout} = {Crin, Rin};
      end
      3'd1:begin
        if(Crin && sign && !A) {Crout, Rout} = {1'b1, 1'b1};
        else                   {Crout, Rout} = {Crin, Rin};
      end
      3'd2:begin
        if(Crin && sign && A)  {Crout, Rout} = {1'b1, 1'b0};
        else                   {Crout, Rout} = {Crin, Rin};
      end
      3'd3:begin
        if(!Crin && sign && A) {Crout, Rout} = {1'b1, 1'b1};
        else                   {Crout, Rout} = {Crin, Rin};
      end
      default:{Crout, Rout} = {Crin, Rin};
    endcase
  end
endtask
endmodule
```

## TestBench

```verilog
`timescale 10ns/10ns
module operations_tb();
reg        clkSys;
reg        rst;
reg        en; 
reg  [3:0] a0;
reg  [3:0] a1;
reg  [3:0] a2;
reg  [3:0] a3; 
reg  [3:0] b0;
reg  [3:0] b1;
reg  [3:0] b2;
reg  [3:0] b3;
reg  [3:0] mode;
wire [3:0] dout0;
wire [3:0] dout1;
wire [3:0] dout2;
wire [3:0] dout3;
wire       ready;
//signed 
operations UUT(
  .clkSys(clkSys), 
  .rst(rst),
  .en(en), 
  .a0(a0), 
  .a1(a1),
  .a2(a2),
  .a3(a3),
  .b0(b0),
  .b1(b1),
  .b2(b2),
  .b3(b3),
  .mode(mode),
  .dout0(dout0),
  .dout1(dout1),
  .dout2(dout2),
  .dout3(dout3),
  .ready(ready)
);

always #10 clkSys = ~clkSys;

initial begin
  clkSys = 0;
  rst = 0;
  a0 = 4'b0110;a1 = 4'b0111;a2 = 4'b1110;a3 = 4'b1011;
  b0 = 4'b1000;b1 = 4'b1010;b2 = 4'b1001;b3 = 4'b0101;
  mode = 0;
  en = 0;
  repeat(2)@(posedge clkSys)rst = 0;
  rst = 1;
  # 20 rst = 0;

  testNOT(4'b1110,4'b0110,4'b1111,4'b0010);
  testNOT(4'b1111,4'b1100,4'b0001,4'b0111);
  
  testAND(4'b1111,4'b1111,4'b1111,4'b0111,4'b1011,4'b1100,4'b0001,4'b0111);
  testAND(4'b1011,4'b1110,4'b0001,4'b0111,4'b1111,4'b1111,4'b1111,4'b0110);
  
  testOR(4'b0001,4'b0010,4'b0100,4'b1000,4'b0010,4'b0100,4'b1000,4'b1000);
  testOR(4'b1001,4'b1010,4'b0001,4'b0000,4'b0110,4'b0101,4'b1110,4'b0000);
  
  testXOR(4'b0101,4'b1010,4'b0100,4'b1000,4'b1010,4'b0101,4'b1111,4'b0000);
  testXOR(4'b1001,4'b1010,4'b0001,4'b0000,4'b0010,4'b0001,4'b1010,4'b0100);
  
  testADDIN(4'b0101,4'b0010,4'b0100,4'b0010,4'b0001,4'b0101,4'b0011,4'b0100);
  testADDIN(4'b1001,4'b1010,4'b0001,4'b0000,4'b0010,4'b0101,4'b1010,4'b0100);
  
  testADDOUT(4'b0101,4'b0010,4'b0100,4'b0010,4'b0001,4'b0101,4'b0011,4'b0100);
  testADDOUT(4'b1001,4'b1010,4'b0001,4'b0000,4'b0010,4'b0101,4'b1010,4'b0100);
  
  testSUBIN(4'b0101,4'b0010,4'b0100,4'b0111,4'b0001,4'b0101,4'b0111,4'b0100);
  testSUBIN(4'b1001,4'b1010,4'b0001,4'b0000,4'b0001,4'b1100,4'b1010,4'b0100);
  
  testSUBOUT(4'b0101,4'b0010,4'b0100,4'b0111,4'b0001,4'b0101,4'b0111,4'b0100);
  testSUBOUT(4'b1001,4'b1010,4'b0001,4'b0001,4'b0001,4'b1100,4'b1010,4'b0110);
  
  testMUL(4'b0011,4'b0011,4'b0010,4'b0000,4'b0010,4'b0011,4'b0001,4'b0010);
  testMUL(4'b0010,4'b0011,4'b0001,4'b0011,4'b0011,4'b0010,4'b0111,4'b0001);
 
  testSMUL(4'b0011,4'b0011,4'b0010,4'b0000,4'b0011,4'b0010,4'b0011,4'b0010);
  testSMUL(4'b0010,4'b0001,4'b0001,4'b0011,4'b0001,4'b0010,4'b0011,4'b0001);
  
  testTWOCOMP(4'b0001,4'b0010,4'b0011,4'b0100,4'b0000,4'b0000,4'b0000,4'b0000);
  testTWOCOMP(4'b1111,4'b1110,4'b1101,4'b1100,4'b0000,4'b0000,4'b0000,4'b0000);
  testTWOCOMP(4'b0100,4'b0101,4'b0110,4'b0111,4'b0000,4'b0000,4'b0000,4'b0000);
  testTWOCOMP(4'b1011,4'b1010,4'b1001,4'b1000,4'b0000,4'b0000,4'b0000,4'b0000);
  
  testABS(4'b0000,4'b0001,4'b0010,4'b0011,4'b0000,4'b0000,4'b0000,4'b0000);
  testABS(4'b0100,4'b0101,4'b0110,4'b0111,4'b0000,4'b0000,4'b0000,4'b0000);
  testABS(4'b1111,4'b1110,4'b1101,4'b1100,4'b0000,4'b0000,4'b0000,4'b0000);
  testABS(4'b1011,4'b1010,4'b1001,4'b1000,4'b0000,4'b0000,4'b0000,4'b0000);
  
  #10000 $stop;
end
//NOT
task testNOT;
  input  [3:0]in0;
  input  [3:0]in1;
  input  [3:0]in2;
  input  [3:0]in3;
  begin
	 #500;
    mode  = 4'd0;
    a0    = in0;
	 a1    = in1;
	 a2    = in2;
	 a3    = in3;
    #20 en = 1;
    #20 en = 0;
	 wait (ready == 1);
	 $display("====== test for function NOT ======");
	 $display("time=%3d A[0]=%b  B[0]=%b   R[0]=%b  ", $time,a0,b0,dout0);
	 $display("time=%3d A[1]=%b  B[1]=%b   R[1]=%b  ", $time,a1,b1,dout1);
	 $display("time=%3d A[2]=%b  B[2]=%b   R[2]=%b  ", $time,a2,b2,dout2);
	 $display("time=%3d A[3]=%b  B[3]=%b   R[3]=%b\n", $time,a3,b3,dout3);
  end
endtask
//AND
task testAND;
  input  [3:0]in0;
  input  [3:0]in1;
  input  [3:0]in2;
  input  [3:0]in3;
  input  [3:0]in4;
  input  [3:0]in5;
  input  [3:0]in6;
  input  [3:0]in7;
  begin
	 #500;
    mode  = 4'd1;
    a0    = in0;
	 a1    = in1;
	 a2    = in2;
	 a3    = in3;
	 b0    = in4;
	 b1    = in5;
	 b2    = in6;
	 b3    = in7;
    #20 en = 1;
    #20 en = 0;
	 wait (ready == 1);
	 $display("====== test for function AND ======");
	 $display("time=%3d A[0]=%b  B[0]=%b   R[0]=%b  ", $time,a0,b0,dout0);
	 $display("time=%3d A[1]=%b  B[1]=%b   R[1]=%b  ", $time,a1,b1,dout1);
	 $display("time=%3d A[2]=%b  B[2]=%b   R[2]=%b  ", $time,a2,b2,dout2);
	 $display("time=%3d A[3]=%b  B[3]=%b   R[3]=%b\n", $time,a3,b3,dout3);
  end
endtask
//OR
task testOR;
  input  [3:0]in0;
  input  [3:0]in1;
  input  [3:0]in2;
  input  [3:0]in3;
  input  [3:0]in4;
  input  [3:0]in5;
  input  [3:0]in6;
  input  [3:0]in7;
  begin
	 #500;
    mode  = 4'd2;
    a0    = in0;
	 a1    = in1;
	 a2    = in2;
	 a3    = in3;
	 b0    = in4;
	 b1    = in5;
	 b2    = in6;
	 b3    = in7;
    #20 en = 1;
    #20 en = 0;
	 wait (ready == 1);
	 $display("====== test for function OR ======");
	 $display("time=%3d A[0]=%b  B[0]=%b   R[0]=%b  ", $time,a0,b0,dout0);
	 $display("time=%3d A[1]=%b  B[1]=%b   R[1]=%b  ", $time,a1,b1,dout1);
	 $display("time=%3d A[2]=%b  B[2]=%b   R[2]=%b  ", $time,a2,b2,dout2);
	 $display("time=%3d A[3]=%b  B[3]=%b   R[3]=%b\n", $time,a3,b3,dout3);
  end
endtask
//XOR
task testXOR;
  input  [3:0]in0;
  input  [3:0]in1;
  input  [3:0]in2;
  input  [3:0]in3;
  input  [3:0]in4;
  input  [3:0]in5;
  input  [3:0]in6;
  input  [3:0]in7;
  begin
	 #500;
    mode  = 4'd3;
    a0    = in0;
	 a1    = in1;
	 a2    = in2;
	 a3    = in3;
	 b0    = in4;
	 b1    = in5;
	 b2    = in6;
	 b3    = in7;
    #20 en = 1;
    #20 en = 0;
	 wait (ready == 1);
	 $display("====== test for function XOR ======");
	 $display("time=%3d A[0]=%b  B[0]=%b   R[0]=%b  ", $time,a0,b0,dout0);
	 $display("time=%3d A[1]=%b  B[1]=%b   R[1]=%b  ", $time,a1,b1,dout1);
	 $display("time=%3d A[2]=%b  B[2]=%b   R[2]=%b  ", $time,a2,b2,dout2);
	 $display("time=%3d A[3]=%b  B[3]=%b   R[3]=%b\n", $time,a3,b3,dout3);
  end
endtask
//ADDIN
task testADDIN;
  input  [3:0]in0;
  input  [3:0]in1;
  input  [3:0]in2;
  input  [3:0]in3;
  input  [3:0]in4;
  input  [3:0]in5;
  input  [3:0]in6;
  input  [3:0]in7;
  begin
	 #500;
    mode  = 4'd4;
    a0    = in0;
	 a1    = in1;
	 a2    = in2;
	 a3    = in3;
	 b0    = in4;
	 b1    = in5;
	 b2    = in6;
	 b3    = in7;
    #20 en = 1;
    #20 en = 0;
	 wait (ready == 1);
	 $display("====== test for function ADDIN ======");
	 $display("time=%3d A[0]=%b  B[0]=%b   tempB[0]=%b  ", $time,a0,b0,UUT.tempB[0]);
	 $display("time=%3d A[1]=%b  B[1]=%b   tempB[1]=%b  ", $time,a1,b1,UUT.tempB[1]);
	 $display("time=%3d A[2]=%b  B[2]=%b   tempB[2]=%b  ", $time,a2,b2,UUT.tempB[2]);
	 $display("time=%3d A[3]=%b  B[3]=%b   tempB[3]=%b\n", $time,a3,b3,UUT.tempB[3]);
  end
endtask
//ADDOUT
task testADDOUT;
  input  [3:0]in0;
  input  [3:0]in1;
  input  [3:0]in2;
  input  [3:0]in3;
  input  [3:0]in4;
  input  [3:0]in5;
  input  [3:0]in6;
  input  [3:0]in7;
  begin
	 #500;
    mode  = 4'd5;
    a0    = in0;
	 a1    = in1;
	 a2    = in2;
	 a3    = in3;
	 b0    = in4;
	 b1    = in5;
	 b2    = in6;
	 b3    = in7;
    #20 en = 1;
    #20 en = 0;
	 wait (ready == 1);
	 $display("====== test for function ADDOUT ======");
	 $display("time=%3d A[0]=%b  B[0]=%b   R[0]=%b  ", $time,a0,b0,dout0);
	 $display("time=%3d A[1]=%b  B[1]=%b   R[1]=%b  ", $time,a1,b1,dout1);
	 $display("time=%3d A[2]=%b  B[2]=%b   R[2]=%b  ", $time,a2,b2,dout2);
	 $display("time=%3d A[3]=%b  B[3]=%b   R[3]=%b\n", $time,a3,b3,dout3);
  end
endtask
//SUBIN
task testSUBIN;
  input  [3:0]in0;
  input  [3:0]in1;
  input  [3:0]in2;
  input  [3:0]in3;
  input  [3:0]in4;
  input  [3:0]in5;
  input  [3:0]in6;
  input  [3:0]in7;
  begin
	 #500;
    mode  = 4'd6;
    a0    = in0;
	 a1    = in1;
	 a2    = in2;
	 a3    = in3;
	 b0    = in4;
	 b1    = in5;
	 b2    = in6;
	 b3    = in7;
    #20 en = 1;
    #20 en = 0;
	 wait (ready == 1);
	 $display("====== test for function SUBIN ======");
	 $display("time=%3d A[0]=%b  B[0]=%b   tempA[0]=%b  ", $time,a0,b0,UUT.tempA[0]);
	 $display("time=%3d A[1]=%b  B[1]=%b   tempA[1]=%b  ", $time,a1,b1,UUT.tempA[1]);
	 $display("time=%3d A[2]=%b  B[2]=%b   tempA[2]=%b  ", $time,a2,b2,UUT.tempA[2]);
	 $display("time=%3d A[3]=%b  B[3]=%b   tempA[3]=%b\n", $time,a3,b3,UUT.tempA[3]);
  end
endtask
//SUBOUT
task testSUBOUT;
  input  [3:0]in0;
  input  [3:0]in1;
  input  [3:0]in2;
  input  [3:0]in3;
  input  [3:0]in4;
  input  [3:0]in5;
  input  [3:0]in6;
  input  [3:0]in7;
  begin
	 #500;
    mode  = 4'd7;
    a0    = in0;
	 a1    = in1;
	 a2    = in2;
	 a3    = in3;
	 b0    = in4;
	 b1    = in5;
	 b2    = in6;
	 b3    = in7;
    #20 en = 1;
    #20 en = 0;
	 wait (ready == 1);
	 $display("====== test for function SUBOUT ======");
	 $display("time=%3d A[0]=%b  B[0]=%b   R[0]=%b  ", $time,a0,b0,dout0);
	 $display("time=%3d A[1]=%b  B[1]=%b   R[1]=%b  ", $time,a1,b1,dout1);
	 $display("time=%3d A[2]=%b  B[2]=%b   R[2]=%b  ", $time,a2,b2,dout2);
	 $display("time=%3d A[3]=%b  B[3]=%b   R[3]=%b\n", $time,a3,b3,dout3);
  end
endtask
//MUL
task testMUL;
  input  [3:0]in0;
  input  [3:0]in1;
  input  [3:0]in2;
  input  [3:0]in3;
  input  [3:0]in4;
  input  [3:0]in5;
  input  [3:0]in6;
  input  [3:0]in7;
  begin
	 #500;
    mode  = 4'd8;
    a0    = in0;
	 a1    = in1;
	 a2    = in2;
	 a3    = in3;
	 b0    = in4;
	 b1    = in5;
	 b2    = in6;
	 b3    = in7;
    #20 en = 1;
    #20 en = 0;
	 wait (ready == 1);
	 $display("====== test for function MUL ======");
	 $display("time=%3d A[0]=%b  B[0]=%b   R[0]=%b  ", $time,a0[1:0],b0[1:0],dout0);
	 $display("time=%3d A[1]=%b  B[1]=%b   R[1]=%b  ", $time,a1[1:0],b1[1:0],dout1);
	 $display("time=%3d A[2]=%b  B[2]=%b   R[2]=%b  ", $time,a2[1:0],b2[1:0],dout2);
	 $display("time=%3d A[3]=%b  B[3]=%b   R[3]=%b\n", $time,a3[1:0],b3[1:0],dout3);
  end
endtask
//SMUL
task testSMUL;
  input  [3:0]in0;
  input  [3:0]in1;
  input  [3:0]in2;
  input  [3:0]in3;
  input  [3:0]in4;
  input  [3:0]in5;
  input  [3:0]in6;
  input  [3:0]in7;
  begin
	 #500;
    mode  = 4'd9;
    a0    = in0;
	 a1    = in1;
	 a2    = in2;
	 a3    = in3;
	 b0    = in4;
	 b1    = in5;
	 b2    = in6;
	 b3    = in7;
    #20 en = 1;
    #20 en = 0;
	 wait (ready == 1);
	 $display("====== test for function SMUL ======");
	 $display("time=%3d A[0]=%b  B[0]=%b   R[0]=%b  ", $time,a0[1:0],b0[1:0],dout0);
	 $display("time=%3d A[1]=%b  B[1]=%b   R[1]=%b  ", $time,a1[1:0],b1[1:0],dout1);
	 $display("time=%3d A[2]=%b  B[2]=%b   R[2]=%b  ", $time,a2[1:0],b2[1:0],dout2);
	 $display("time=%3d A[3]=%b  B[3]=%b   R[3]=%b\n", $time,a3[1:0],b3[1:0],dout3);
  end
endtask
//TWOCOMP
task testTWOCOMP;
  input  [3:0]in0;
  input  [3:0]in1;
  input  [3:0]in2;
  input  [3:0]in3;
  input  [3:0]in4;
  input  [3:0]in5;
  input  [3:0]in6;
  input  [3:0]in7;
  begin
	 #500;
    mode  = 4'd10;
    a0    = in0;
	 a1    = in1;
	 a2    = in2;
	 a3    = in3;
	 b0    = in4;
	 b1    = in5;
	 b2    = in6;
	 b3    = in7;
    #20 en = 1;
    #20 en = 0;
	 wait (ready == 1);
	 $display("====== test for function TWOCOMP ======");
	 $display("time=%3d A[0]=%b  B[0]=%b   R[0]=%b  ", $time,a0,b0,dout0);
	 $display("time=%3d A[1]=%b  B[1]=%b   R[1]=%b  ", $time,a1,b1,dout1);
	 $display("time=%3d A[2]=%b  B[2]=%b   R[2]=%b  ", $time,a2,b2,dout2);
	 $display("time=%3d A[3]=%b  B[3]=%b   R[3]=%b\n", $time,a3,b3,dout3);
  end
endtask
//ABS
task testABS;
  input  [3:0]in0;
  input  [3:0]in1;
  input  [3:0]in2;
  input  [3:0]in3;
  input  [3:0]in4;
  input  [3:0]in5;
  input  [3:0]in6;
  input  [3:0]in7;
  begin
	 #500;
    mode  = 4'd11;
    a0    = in0;
	 a1    = in1;
	 a2    = in2;
	 a3    = in3;
	 b0    = in4;
	 b1    = in5;
	 b2    = in6;
	 b3    = in7;
    #20 en = 1;
    #20 en = 0;
	 wait (ready == 1);
	 $display("====== test for function ABS ======");
	 $display("time=%3d A[0]=%b  B[0]=%b   R[0]=%b  ", $time,a0,b0,dout0);
	 $display("time=%3d A[1]=%b  B[1]=%b   R[1]=%b  ", $time,a1,b1,dout1);
	 $display("time=%3d A[2]=%b  B[2]=%b   R[2]=%b  ", $time,a2,b2,dout2);
	 $display("time=%3d A[3]=%b  B[3]=%b   R[3]=%b\n", $time,a3,b3,dout3);
  end
endtask
endmodule
```

## Monitor

![operations(add%20SMUL)%203d8e0711608b49f99dc639b151f1888b/Untitled.png](operations(add%20SMUL)%203d8e0711608b49f99dc639b151f1888b/Untitled.png)

![operations(add%20SMUL)%203d8e0711608b49f99dc639b151f1888b/Untitled%201.png](operations(add%20SMUL)%203d8e0711608b49f99dc639b151f1888b/Untitled%201.png)

![operations(add%20SMUL)%203d8e0711608b49f99dc639b151f1888b/Untitled%202.png](operations(add%20SMUL)%203d8e0711608b49f99dc639b151f1888b/Untitled%202.png)

![operations(add%20SMUL)%203d8e0711608b49f99dc639b151f1888b/Untitled%203.png](operations(add%20SMUL)%203d8e0711608b49f99dc639b151f1888b/Untitled%203.png)

## Wave

![operations(add%20SMUL)%203d8e0711608b49f99dc639b151f1888b/Untitled%204.png](operations(add%20SMUL)%203d8e0711608b49f99dc639b151f1888b/Untitled%204.png)

![operations(add%20SMUL)%203d8e0711608b49f99dc639b151f1888b/Untitled%205.png](operations(add%20SMUL)%203d8e0711608b49f99dc639b151f1888b/Untitled%205.png)

## Compilation Report

![operations(add%20SMUL)%203d8e0711608b49f99dc639b151f1888b/Untitled%206.png](operations(add%20SMUL)%203d8e0711608b49f99dc639b151f1888b/Untitled%206.png)

## State Machine

![operations(add%20SMUL)%203d8e0711608b49f99dc639b151f1888b/Untitled%207.png](operations(add%20SMUL)%203d8e0711608b49f99dc639b151f1888b/Untitled%207.png)

## Fmax

![operations(add%20SMUL)%203d8e0711608b49f99dc639b151f1888b/Untitled%208.png](operations(add%20SMUL)%203d8e0711608b49f99dc639b151f1888b/Untitled%208.png)

![operations(add%20SMUL)%203d8e0711608b49f99dc639b151f1888b/9C47687C-CAA1-44C3-8A2A-A2D3AF1C87B3.jpeg](operations(add%20SMUL)%203d8e0711608b49f99dc639b151f1888b/9C47687C-CAA1-44C3-8A2A-A2D3AF1C87B3.jpeg)