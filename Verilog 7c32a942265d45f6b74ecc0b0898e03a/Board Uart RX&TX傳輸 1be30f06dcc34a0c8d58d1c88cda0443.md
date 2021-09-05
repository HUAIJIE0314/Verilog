# Board Uart RX&TX傳輸

## usage_Uart_RX

```verilog
//`include "Uart_RX.v"
	//`include "usage_Uart_TX"
	//`include "edge_detect.v"
	module usage_Uart_RX(TX_D, RX_D, clk, rst_n, busy);
	//ports declartaion
	input       clk, rst_n, RX_D;
	output TX_D;
	wire [7:0]Tx_data, Rx_data;
	output wire busy;
	wire pos_edge, neg_edge, finish;
	assign Tx_data = Rx_data;
	
	Uart_RX U0(
		.clk_sys(clk),
		.rst_n(rst_n), 
		.RX_D(RX_D),
		.busy(busy),
		.dout(Rx_data),
		//.samp(samp)
		.finish(finish)
	);
	/*
	Uart_TX_ U1(
		.clk_sys(clk),
		.rst_n(rst_n),
		.en(pos_edge),
		.din(Tx_data),
		.busy(),
		.TX_D(TX_D)
	);
	*/
	
	usage_Uart_TX U1(
		.clk_50M(clk),
		.reset_n(rst_n),
		.write(pos_edge),
		.write_value(Tx_data),
		.uart_txd(TX_D),
		.busy()
	);
	
	edge_detect U2(
		.clk(clk),
		.rst_n(rst_n),
		.data_in(busy),//busy
		.pos_edge(pos_edge),
		.neg_edge(neg_edge) 
   );
	/*
	always@(posedge clk or negedge rst_n)begin
		if(!rst_n)begin
			led_out <= 4'b0000;
		end
		else begin
			if(TX_data==8'h41)begin
				led_out <= 4'b1000;
			end
			else begin
				led_out <= 4'b1111;
			end
		end
	end
	*/
	endmodule
```

都可達到100%資訊正確！