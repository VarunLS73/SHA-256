# SHA-256
Implementation of SHA-256
module sha256(input_data, input_valid, input_ready, last_word, clk, rst, output_valid, hash_data);

// Registers and control signals for SHA-256 processing
reg [4:0] length_counter;     
reg [63:0] size;              
reg [1:0] special;            
reg last_word_delayed;        
reg [1:0] last_next;          

// Reset logic - Initializes control registers
always @(negedge rst)
begin
    input_ready <= 1'b1;      
    length_counter <= 5'd31;  
    special <= 2'd0;          
    size <= 64'hffffffffffffffe0;
    last_word_delayed <= 1'b0;
end

// Instantiation of SHA-256 processing engine
sha_engine Eng(output_data, clk, block_counter, rst, output_valid, hash_data, last_word, last_next);

// Input handling - Stops accepting input at 14th word
always @(posedge clk)
begin
    if(input_valid && input_ready)
    begin
        if(length_counter == 5'd14)
            input_ready <= 1'b0;
        length_counter = length_counter + 1;
    end
end

// Delay last_word signal for padding logic
always @(posedge clk)
begin
    if(last_word)
        last_word_delayed <= 1'b1;
end

// Determines padding strategy based on message length
always @(posedge last_word_delayed)
begin
    if(length_counter <= 13)
        special <= 2'b01;
    else
        special <= 2'b11;
end

// Data processing and padding logic
always @(posedge clk)
begin
    case(special)
        2'b00 : output_data <= input_data;  // Normal data processing
        2'b01 : last_next <= 2'b01;         // First padding block
        2'b11 : last_next <= 2'b10;         // Final padding block
    endcase
end

// Reset counters after block processing
always @(posedge clk)
begin
    if(block_counter == 64)
    begin
        length_counter <= 5'd31;
        input_ready <= 1'b1;
    end
end

// Update total message size
always @(posedge clk)
begin
    if(input_ready && special == 0)
        size <= size + 32;
end

endmodule

