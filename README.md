# 4-stage-pipelined-proceesor
module ppp(


    input wire clk
);

    // Define instruction and data widths
    parameter INSTR_WIDTH = 16;
    parameter REG_WIDTH = 8;
    parameter ADDR_WIDTH = 4;

    // Register file: 16 registers, 8-bit wide
    reg [REG_WIDTH-1:0] registers [0:15];
    
    // Simple instruction memory (16 instructions max)
    reg [INSTR_WIDTH-1:0] instr_mem [0:15];

    // Simple data memory (256 bytes)
    reg [REG_WIDTH-1:0] data_mem [0:255];

    // IF/ID pipeline register
    reg [INSTR_WIDTH-1:0] IF_ID;

    // ID/EX pipeline register: {opcode[3:0], rd[3:0], rs1_val[7:0], rs2_or_imm[7:0]}
    reg [31:0] ID_EX;

    // EX/MEM pipeline register: {opcode[3:0], rd[3:0], result[7:0]}
    reg [15:0] EX_MEM;

    // Opcodes
    localparam OPCODE_ADD  = 4'b0001;
    localparam OPCODE_SUB  = 4'b0010;
    localparam OPCODE_LOAD = 4'b0011;

    // Fetch stage
    reg [ADDR_WIDTH-1:0] pc;
    always @(posedge clk) begin
        IF_ID <= instr_mem[pc];
        pc <= pc + 1;
    end

    // Decode stage
    wire [3:0] opcode, rd, rs1, rs2imm;
    assign opcode = IF_ID[15:12];
    assign rd     = IF_ID[11:8];
    assign rs1    = IF_ID[7:4];
    assign rs2imm = IF_ID[3:0];

    wire [REG_WIDTH-1:0] rs1_val = registers[rs1];
    wire [REG_WIDTH-1:0] rs2_val = registers[rs2imm]; // if used as register
    wire [REG_WIDTH-1:0] imm_val = {4'b0000, rs2imm}; // zero-extend immediate

    always @(posedge clk) begin
        ID_EX <= {opcode, rd, rs1_val, (opcode == OPCODE_LOAD) ? imm_val : rs2_val};
    end

    // Execute stage
    wire [3:0] ex_opcode = ID_EX[31:28];
    wire [3:0] ex_rd     = ID_EX[27:24];
    wire [REG_WIDTH-1:0] ex_rs1_val = ID_EX[23:16];
    wire [REG_WIDTH-1:0] ex_rs2_val = ID_EX[15:8];
    reg [REG_WIDTH-1:0] ex_result;

    always @(posedge clk) begin
        case (ex_opcode)
            OPCODE_ADD:  ex_result <= ex_rs1_val + ex_rs2_val;
            OPCODE_SUB:  ex_result <= ex_rs1_val - ex_rs2_val;
            OPCODE_LOAD: ex_result <= data_mem[ex_rs2_val];  // Use rs2_val as address
            default:     ex_result <= 8'h00;
        endcase
        EX_MEM <= {ex_opcode, ex_rd, ex_result};
    end

    // Writeback stage
    wire [3:0] wb_opcode = EX_MEM[15:12];
    wire [3:0] wb_rd     = EX_MEM[11:8];
    wire [REG_WIDTH-1:0] wb_result = EX_MEM[7:0];

    always @(posedge clk) begin
        if (wb_opcode == OPCODE_ADD || wb_opcode == OPCODE_SUB || wb_opcode == OPCODE_LOAD)
            registers[wb_rd] <= wb_result;
    end

endmodule
