# segment_improve

```verilog
module segment_improve(clk_sys, rst, data1, data2, data3, data4);
	/*----------in/output-----------*/
	input clk_sys, rst;
	output reg [6:0]data1, data2, data3, data4;
	/*----------variables-----------*/
	reg [3:0]hhhh, hhh, hh, h;
	reg [25:0]cnt_speed = 26'd0;
	/*----------make counter-----------*/
	always@(posedge clk_sys or negedge rst)begin
		if(!rst)cnt_speed <= 26'd0;
		else cnt_speed <= (cnt_speed<26'd50000000)?(cnt_speed + 26'd1):(26'd0);
	end
	/*----------seven_seg-----------*/
	always@(posedge clk_sys or negedge rst)begin
		if(!rst)begin
			data1 <= 7'd0;
			data2 <= 7'd0;
			data3 <= 7'd0;
			data4 <= 7'd0;
		end
		else begin
			B_to_s(hhhh,data4);
			B_to_s(hhh ,data3);
			B_to_s(hh  ,data2);
			B_to_s(h   ,data1);
		end
	end
	always@(posedge clk_sys or negedge rst)begin
		if(!rst)begin
			hhhh <= 4'd0;
			hhh  <= 4'd0;
			hh   <= 4'd0;
			h    <= 4'd0;
		end
		else begin
			if(cnt_speed==26'd49999999)begin
				if(h>=4'd9)begin
					h <= 4'd0;
					if(hh>=4'd9)begin
						hh <= 4'd0;
						if(hhh>=4'd9)begin
							hhh <= 4'd0;
							if(hhhh>=4'd9)hhhh <= 4'd0;
							else hhhh = hhhh + 4'd1;
						end
						else hhh <= hhh + 4'd1;
					end
					else hh <= hh + 4'd1;
				end
				else h <= h + 4'd1;
			end
		end
	end
	task B_to_s(input [3:0]in, output reg [6:0]out);
		case(in)
			4'd0:out = 7'b0000001;
			4'd1:out = 7'b1001111;
			4'd2:out = 7'b0010010;
			4'd3:out = 7'b0000110;
			4'd4:out = 7'b1001100;
			4'd5:out = 7'b0100100;
			4'd6:out = 7'b0100000;
			4'd7:out = 7'b0001111;
			4'd8:out = 7'b0000000;
			4'd9:out = 7'b0001100;
			default:out = 7'b1111110;
		endcase
	endtask
	
	endmodule
```