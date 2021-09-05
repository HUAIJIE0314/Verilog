# LCD Library(同步式設計)

## LCD

```verilog
	module LCD(clk_sys, rst_n, rs, rw, en, data_in, lcd_on, BL, rotate, tick_newdata, tick_1khz, tick_1hz,
				  data0,  data1,  data2,  data3,  data4,  data5,  data6,  data7,  data8,  data9,
				  data10, data11, data12, data13, data14, data15, data16, data17, data18, data19, 
				  data20, data21, data22, data23, data24, data25, data26, data27, data28, data29,
				  data30, data31, data32, data33, data34, data35, data36, data37, data38, data39
				  );
	/*-----parameter-----*/
	parameter ins_function    = 3'd0;
	parameter ins_effect      = 3'd1;
	parameter ins_mode_setup  = 3'd2;
	parameter ins_clear       = 3'd3;
	parameter ins_up_row      = 3'd4;
	parameter ins_low_row     = 3'd5;
	parameter data_din        = 3'd6;
	parameter loop            = 3'd7;
	/*-----in/output-----*/
	input clk_sys, rst_n, rotate, tick_newdata, tick_1khz, tick_1hz;
	input [7:0]data0,  data1,  data2,  data3,  data4,  data5,  data6,  data7,  data8,  data9,
				  data10, data11, data12, data13, data14, data15, data16, data17, data18, data19, 
				  data20, data21, data22, data23, data24, data25, data26, data27, data28, data29,
				  data30, data31, data32, data33, data34, data35, data36, data37, data38, data39;
	output en, rs, rw, data_in, lcd_on, BL;
	reg rs;
	reg [7:0]data_in;
	assign rw = 1'b0, lcd_on = 1'b1, BL = 1'b1;
	/*-----variables-----*/
	wire[7:0] data_temp[39:0];
	reg [5:0] index;//~39
	reg [2:0] fstate;
	/*-----assign wire-----*/
	assign en = (rotate)?((&(fstate))?(tick_1hz):(tick_1khz)):((fstate<=data_din)?(tick_1khz):(1'b0));
	/*-----lcd_data-----*/
	assign data_temp[0]  = data0;
	assign data_temp[1]  = data1;
	assign data_temp[2]  = data2;
	assign data_temp[3]  = data3;
	assign data_temp[4]  = data4;
	assign data_temp[5]  = data5;
	assign data_temp[6]  = data6;
	assign data_temp[7]  = data7;
	assign data_temp[8]  = data8;
	assign data_temp[9]  = data9;
	assign data_temp[10] = data10;
	assign data_temp[11] = data11;
	assign data_temp[12] = data12;
	assign data_temp[13] = data13;
	assign data_temp[14] = data14;
	assign data_temp[15] = data15;
	assign data_temp[16] = data16;
	assign data_temp[17] = data17;
	assign data_temp[18] = data18;
	assign data_temp[19] = data19;
	assign data_temp[20] = data20;
	assign data_temp[21] = data21;
	assign data_temp[22] = data22;
	assign data_temp[23] = data23;
	assign data_temp[24] = data24;
	assign data_temp[25] = data25;
	assign data_temp[26] = data26;
	assign data_temp[27] = data27;
	assign data_temp[28] = data28;
	assign data_temp[29] = data29;
	assign data_temp[30] = data30;
	assign data_temp[31] = data31;
	assign data_temp[32] = data32;
	assign data_temp[33] = data33;
	assign data_temp[34] = data34;
	assign data_temp[35] = data35;
	assign data_temp[36] = data36;
	assign data_temp[37] = data37;
	assign data_temp[38] = data38;
	assign data_temp[39] = data39;
	/*------------------fstate-----------------*/
	/*-----state-----*/
	always@(posedge clk_sys or negedge rst_n)begin
		if(!rst_n)fstate <= ins_function;
		else begin
			case(fstate)
				ins_function:begin
					if(tick_1khz)fstate <= ins_effect;
					else         fstate <= ins_function;
				end 
				ins_effect:begin
					if(tick_1khz)fstate <= ins_mode_setup;
					else         fstate <= ins_effect;
				end	
				ins_mode_setup:begin
					if(tick_1khz)fstate <= ins_clear;
					else         fstate <= ins_mode_setup;
				end 
				ins_clear:begin
					if(tick_1khz)fstate <= ins_up_row;
					else         fstate <= ins_clear;
				end
				ins_up_row:begin
					if(tick_1khz)fstate <= data_din;
					else         fstate <= ins_up_row;
				end 
				data_din:begin
					if(index==6'd19&&tick_1khz)     fstate <= ins_low_row;
					else if(index==6'd39&&tick_1khz)fstate <= loop;
					else                            fstate <= data_din;
				end
				ins_low_row:begin
					if(tick_1khz)fstate <= data_din;
					else         fstate <= ins_low_row;
				end 
				loop:begin
					if(tick_newdata)fstate <= ins_up_row;
					else            fstate <= loop;
				end 
					
				default:fstate <= ins_function;
					
			endcase
		end
	end
	/*-----output-----*/
	always@(posedge clk_sys)begin
		if(tick_1khz)begin
			case(fstate)
				ins_function:begin
					rs      <= 1'b0;
					data_in <= 8'h38;
				end
					ins_effect:begin
					rs      <= 1'b0;
					data_in <= 8'h0c;
				end
				ins_mode_setup:begin
					rs      <= 1'b0;
					data_in <= 8'h06;
				end
				ins_clear:begin
					rs      <= 1'b0;
					data_in <= 8'h01;
				end
				ins_up_row:begin
					rs      <= 1'b0;
					data_in <= 8'h80;
				end
				data_din:begin
					rs      <= 1'b1;
					data_in <= data_temp[index];
					if(index==6'd39)index <= 6'd0;
					else            index <= index + 6'd1;
				end
				ins_low_row:begin
					rs      <= 1'b0;
					data_in <= 8'hc0;
				end
				loop:begin
					if(rotate)begin
						rs      <= 1'b0;
						data_in <= 8'h18;
					end
					else begin
						rs      <= rs;
						data_in <= data_in;
					end
				end
				default:begin
					rs      <= 1'b0;
					data_in <= 8'h01;
				end
			endcase
		end
		else begin
			rs      <= rs;
			data_in <= data_in;
		end
	end
	endmodule
```