# Day3 Verilog 資料型態(上)

```verilog
## 資料型態
|  值  |   意義    |
|:---:|:----------:|
| `0` | 低電位(邏輯0)  
| `1` | 高電位(邏輯1)  
| `Z` | 高阻抗( High Impendence ) 
| `X` | 未知的值(Unknow)or邏輯衝突)

## 連接線：wire、wand、wor
- 沒有記憶性
- 預設值為`z`
- 將兩個wire連在一起是不允許的
- 若是型態為wand/wor則例外
舉個例子：
```
module test(a, b, m, n);
    input a;
    input b;
    output m;
    output n;
    wand m;
    wor n;
    
    // wire and ---> m = a&b
    assign m = a;
    assign m = b;

    // wire or ---> n = a|b
    assign n = a;
    assign n = b;

endmodule
```

## 暫存器：reg
- 有記憶性
- 預設值為x (最好要初始化，通常使用rst或rst_n訊號觸發初始化)
舉個例子：
```
module test(clk, rst_n, a, b);
    input clk;
    input a;
    input rst_n;
    output b;
    reg b;
    
    always@(posedge clk or negedge rst_n)begin
        if(!rst_n)b <= 1'b0;
        else      b <= a;
    end
    
endmodule
```

到這邊後，應該有些人對於reg與wire的使用不是很理解，先來解釋位甚麼第二個例子需要用reg型態，`因為變數b在alwaye內賦值，而always又是屬於觸發型電路，所以需要用暫存器儲存<前態>與<次態>，是不可用wire的(wire沒有記憶性)`，順帶一提，這邊的reset如果是負緣，那我們通常會加個_n，讓別人知道那是負緣觸發，若是正緣的話則不用加，這樣的coding-style是比較好的！！
```