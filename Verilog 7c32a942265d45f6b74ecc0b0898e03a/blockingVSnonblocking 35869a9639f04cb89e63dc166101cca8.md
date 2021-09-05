# blockingVSnonblocking

```verilog
module blockingVSnonblocking(
  clkSys, 
  rst_n,
  a_out,
  b_out,
  c_out
);

parameter width = 3;

function integer log2;
  input integer in ;
  for(log2=0; in>1; log2=log2+1) begin
    in = in >> 1 ;
  end
endfunction

input clkSys;
input rst_n;

output [log2(width):0]a_out;
output [log2(width):0]b_out;
output [log2(width):0]c_out;

reg [log2(width):0]a;
reg [log2(width):0]b;
reg [log2(width):0]c;

assign {a_out, b_out, c_out} = {a, b, c};

always@(posedge clkSys or negedge rst_n)begin
  if(!rst_n)begin
    a <= 2'd1;
	 b <= 2'd2;
	 c <= 2'd3;
  end
  else begin
    a <= c;
	 b <= a;
	 c <= b;
  end
end

endmodule
```

![Untitled](blockingVSnonblocking%2035869a9639f04cb89e63dc166101cca8/Untitled.png)