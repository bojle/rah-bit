# RAH Protocol User Guide

## Overview

The RAH (Real-time Application Handler) protocol is designed to facilitate the transfer of data between a CPU and an FPGA, which is developed by [Vicharak](https://vicharak.in), And It allows the CPU to run various applications that send data to the FPGA. The RAH Services encapsulate the data into distinguishable data-frames identified by an `app_id` and deliver these frames to the FPGA. On the FPGA side, the RAH design decodes the data and writes it to the corresponding `APP_WR_FIFO`. Similarly, during the read cycle, the FPGA writes data into `APP_RD_FIFO`, which is then encapsulated and delivered back to the CPU where it gets decoded.

This guide includes the capability to generate and manage multiple applications on both the CPU and FPGA using header files.

## Block Diagram

<div align="center">

![rah design](images/rah_user_guide.svg)

</div>

## Key Components

1. **CPU Applications**: These applications generate data that needs to be sent to the FPGA.
2. **RAH Services**: Encapsulates and decapsulates data frames based on the `app_id`.
3. **FPGA RAH Design**: Handles the decoding of incoming data frames and writes to the appropriate FIFO.
4. **APP_WR_FIFO**: FIFO buffer where decoded data from CPU is written.
5. **APP_RD_FIFO**: FIFO buffer where data generated by FPGA applications is written before being sent to the CPU.

## Data Transfer Process

### Write Cycle (CPU to FPGA)

1. **CPU Application**: Generates data to be sent to the FPGA.
2. **RAH Services on CPU**: Encapsulates the data into a data-frame, including the `app_id`.
3. **RAH Design on FPGA**: Receives the data-frame and decodes it.
4. **FPGA Application**: Reads the decoded data from the appropriate `APP_WR_FIFO`.

### Read Cycle (FPGA to CPU)

1. **FPGA Application**: Writes data to the `APP_RD_FIFO`.
2. **RAH Design on FPGA**: Encapsulates the data from `APP_RD_FIFO` into a data-frame.
3. **RAH Services on CPU**: Receives the data-frame and decodes it.
4. **CPU Application**: Processes the received data.

## Detailed Instructions

### Writing Data from CPU to FPGA

1. **CPU Application**:
    - Generate the data you wish to send to the FPGA.
    - Identify the `app_id` associated with this data.
    - The data width of the encapsulated data-frame is 48-bit (6-bytes).Send the data in multiple of 6-bytes, if the data is not in multiple of 6-bytes then append it with psuedo data bytes and access the data bytes at FPGA side using index of the data-bytes.

2. **Encapsulation by RAH Services**:
    - Call the RAH API to encapsulate the data. The API function might look like `rah_write(app_id, data, length)`. Where `length` is number of bytes.
    - Ensure that the data is correctly encapsulated into a data-frame with the specified `app_id`.
    - The data will be encapsulated as 6-bytes per data-line of data-frame.

3. **Transmission**:
    - The RAH Services will transmit the data-frame to the FPGA.

4. **Decoding by RAH Design on FPGA**:
    - The FPGA RAH design will receive the data-frame and decode it.
    - The decoded data will be written to the corresponding `APP_WR_FIFO`.

5. **Reading from `APP_WR_FIFO`**:
    - FPGA applications can request data from the FIFO using a function like `fifo_read(app_id)`.
    - The data associated with the `app_id` will be available for the application to process.
    - This is a 48-bit FIFO.

### Reading Data from FPGA to CPU

1. **FPGA Application**:
    - Generate the data you wish to send to the CPU.
    - Write this data to the `APP_RD_FIFO` using APP_RD_FIFO_EN signal.
    - This is a 48-bit FIFO.
    - Write data in FIFO in multiple of 6-bytes and is data is not multiple of 6-bytes then append it with psuedo data bytes and access the data at CPU side using index of data bytes.

2. **Encapsulation by RAH Design on FPGA**:
    - The RAH design will read the data from `APP_RD_FIFO`.
    - The data will be encapsulated into a data-frame with the specified `app_id`.

3. **Transmission**:
    - The data-frame will be transmitted to the CPU by the RAH design.

4. **Decapsulation by RAH Services on CPU**:
    - The CPU RAH Services will receive the data-frame and decapsulate it.
    - The decapsulated data will be made available to the CPU application.

5. **Processing by CPU Application**:
    - The CPU application can access the received data using a function like `rah_read(app_id)`.
    - The data associated with the `app_id` will be available for the application to process.
    - The data will be received as 6-bytes per data-line.

> **_NOTE:_** The allignment of the data to be send and recieve through out the RAH protocol is user defined. The user have to make sure that the data is sampled in the same way as it is alligned at the time of transmission. This valids for both (write and read) cycles.

## Generating Multiple Applications

### Using Header Files

To manage multiple applications on both the CPU and FPGA, you can define and include header files that specify the application identifiers (`app_id`) and related configurations.

#### CPU Side

1. **Define Applications and Use Application IDs in CPU Code**:

    ```c
    // main.c
    #include <rah.h>

    #define APP_ID_1 1
    #define APP_ID_2 2
    #define APP_ID_3 3

    void app1() {
        char data[] = {0x01, 0x02, 0x03, 0x04, 0x05, 0x06};
        rah_write(APP_ID_1, data, 6);
    }

    void app2() {
        char data[] = {
            0x00, 0x01, 0x00, 0x05, 0x05, 0x02,
            0x70, 0x01, 0x03, 0x04, 0x05, 0x02
        };
        rah_write(APP_ID_2, data, sizeof(data));
    }

    void app3() {
        char data[] = {
            0x90, 0x01, 0x25, 0x05, 0x34, 0x02,
            0x70, 0x01, 0x03, 0x04, 0x05, 0x02,
            0x70, 0x67, 0x03, 0x78, 0x05, 0x65
        };
        rah_write(APP_ID_3, &data, sizeof(data));
    }

    int main() {
        app1();
        app2();
        app3();
        return 0;
    }
    ```

    - Installation and running
    ```C
    //Compilation
    gcc filename.c -lrah

    //Running the executable
    ./a.out
    ```

#### FPGA Side

1. **Define Applications in a Header File (rah_var_defs.vh)**:
    ```verilog
    // rah_var_defs.vh
    `define TOTAL_APPS 3

    `define APP_ID_1 1
    `define APP_ID_2 2
    `define APP_ID_3 3
    // Add more application IDs as needed

    `define GET_DATA_RAH(a) rd_data[a * RAH_PACKET_WIDTH +: RAH_PACKET_WIDTH]
    `define SET_DATA_RAH(a) wr_data[a * RAH_PACKET_WIDTH +: RAH_PACKET_WIDTH]
    ```

2. **Include and Use Application IDs in FPGA Code**:
    ```verilog
    // top.v
    `include "rah_var_defs.vh"

    module top (
        ..........
    );
    
    /* Accesssing data from APP_WR_FIFO */

    assign rd_clk[`APP_ID_1] = rd_clk_app1; 

    app1_recv #(
        .RAH_PACKET_WIDTH(RAH_PACKET_WIDTH)
    ) er (
        .clk                    (rd_clk_app1),
        .data_queue_empty       (data_queue_empty[`APP_ID_1]),
        .data_queue_almost_empty(data_queue_almost_empty[`APP_ID_1]),
        .request_data           (request_data[`APP_ID_1]),
        .data_frame             (`GET_DATA_RAH(`APP_ID_1)),
        .uart_tx_pin            (uart_tx_pin)
    );

    /* Writing data into APP_RD_FIFO */
    
    assign wr_clk[`APP_ID_1] = wr_clk_app1;

    /* Include your module */
    app1_trans #(
        .RAH_PACKET_WIDTH(RAH_PACKET_WIDTH)
    ) et (
        .clk        (wr_clk_app1),
        .data       (`SET_DATA_RAH(`APP_ID_1)),
        .send_data  (write_apps_data[`APP_ID_1])
    );


    // Add Applications as needed.

    endmodule
    ```
> [!NOTE]
>
> The FIFOs used here for data transfer are asynchronous. So, it is mandatory to assign the read and write clocks for every application FIFO. It facilitates us Clock Domain Crossing (CDC) among RAH and different applications.

## Requesting Data from FIFO

### On FPGA Side

- **Read from `APP_WR_FIFO`**:
  ```verilog
  reg [1:0] state = 0;
  localparam IDLE = 2'b00;
  localparam WAIT = 2'b01;
  localparam READ = 2'b10;

  always @(posedge clk) begin
    case (state)

    IDLE: begin
        if (!APP_WR_FIFO_empty_1) begin
            APP_WR_FIFO_EN_1 <= 1'b1;
            state <= WAIT;
        end
    end
    
    WAIT: begin
        APP_WR_FIFO_EN_1 <= 0;
        state <= READ;
    end

    READ: begin
        READ_DATA <= APP_WR_FIFO_READ_DATA_1 ;
        state <= IDLE;
    end

    default : state <= IDLE;

    endcase
  end
    ```
- **Check if `APP_WR_FIFO` is empty.**
- **Assert FIFO Enable signal.**
- **Wait for one clock period.**
- **Sample the data.**
- **Read data from FIFO using the read pointer.**

### On CPU Side

- **Read from `APP_RD_FIFO`:**

    ```c
    char data[6];
    rah_read(app_id, data, 6); // 6 is length of the data
    ```
- **Call `rah_read(app_id)` to get the data associated with `app_id`.**

## Writing Data to FIFO

### On FPGA Side

- **Write to `APP_RD_FIFO`.**:
    ```verilog
    always @(posedge clk) begin
        case(state)
        ....

        WRITE: begin
            APP_RD_FIFO_EN <= 1;
            APP_RD_FIFO_DATA <= DATA;
        end

        ....
    
    end
    ```
- **Assert high the Enable signals of FIFO.**
- **Write data to FIFO at the same clk edge.**

### On CPU Side

- **Write to `APP_WR_FIFO`**:
    ```c
    char data[] = {0x01, 0x02, 0x03, 0x04, 0x05, 0x06};
    rah_write(app_id, data, sizeof(data));
    ```
- **Call `rah_write(app_id, data, length)` to encapsulate and send data to FPGA.**
- **`length` is the number of bytes.**

---
This enhanced guide covers the essentials of using the RAH protocol for data transfer between a CPU and FPGA, including the management of multiple applications using header files. For more detailed API functions and specific implementation details, refer to the RAH protocol's technical documentation.