---
title: FPGA DDS Walkthrough
parent: Projects
nav_order: 1
---

# FPGA DDS Walkthrough

This tutorial describes how to build a direct digital synthesis (DDS) using Verilog and a lookup table.  The `phase_step` which controls the frequency of the DDS is contained in an AXI lite component allowing adjustment at runtime.  A 

A low cost AMD/Xilinx Zynq 7 (Zybo legacy) board is used for prototyping.  A Digilent DA3 DAC is connected to a PMOD output on the board.  

A Saleae logic analyzer is used for probing digital signals on a PMOD header from the Zynq 7 board.

The sources for this project can be found in the [fpga-tutorials repo](https://github.com/shraken/fpga-tutorials).

## Requirements

This project used Vivado and Petalinux 2018.2 toolchain.  This was selected because Digilent had previously produced a 2017.4 image [here](https://github.com/Digilent/Petalinux-Zybo)

The OS for the petalinux uses ubuntu 16.04 LTS and a Vagrantfile is provided.  

## Architecture

- Phase accumulator
- Sine LUT
- SPI master for DAC data

### DDS IP with Phase Accumulator

The `dds_dac_core` was created as an IP block.  The `Tools -> Create and package IP` menu option is used.  Package it as an AXI4 Peripheral.

This tutorial from Digilent is helpful to explain [usage](https://digilent.com/reference/learn/programmable-logic/tutorials/zybo-creating-custom-ip-cores/start).

#### Debugging

The `(* MARK_DEBUG = "true" *)` is used to indicate a register should be exposed as a debug net.  You can follow this guide [here](https://vhdlwhiz.com/using-ila-and-vio/) for setting up Virtual Input/Output (VIO) and Integrated Logc Analyzer (ILA) in your project.  

With that marking you can click `Run Synthesis`.  Then click the `Set Up Debug` option.  

![Setup debug after synthesis](/assets/images/fpga-dds/setup_debug.png)

You can pick the nets that have been marked with the `MARK_DEBUG` text from above.

![Debug nets visible from design](/assets/images/fpga-dds/debug_nets.png)

#### Sine Lut

The sine LUT is implemented as hard code table in the Verilog.  Initially, I used the `readmemh` but I couldn't figure out how to get the .hex file to get picked up by the Vivado build.  Using `readmemh` to read into the `sine_lut` regisiter memory worked fine with a top level project but not when packaged as an IP. 

You can use the `sine wave generate` jupyter notebook to create a `hex` file or generate the contents to be copied into the `dds` module.

The `dds` module implements the direct digital synthesis that uses the phase accumulator and sine LUT table.  The inputs and outputs:

```verilog
module dds#(
    parameter PHASE_WIDTH = 32,     // width of phase accumulator
    parameter ADDR_WIDTH = 8        // 256-entry LUT
)(
    input wire clk,
    input wire reset,
    input wire [PHASE_WIDTH-1:0] phase_step,
    
    (* MARK_DEBUG = "true" *)
    output reg [7:0] dds_sample
    );
```

Tjhe `dds` module operates with a simple design.  A register with a bit width of `PHASE_WIDTH` is used to accumulate number of ticks in the `phase_acc` register.  When the user asserts the `reset` in the control register then the `phase_acc` is set to 0 value.  If the `reset` is not-asserted then the `phase_acc` is increased by the control register user register setting `phase_step` on each `clk` tick.

The upper bits of the `phase_acc` are used as index and saved to `lut_addr`.  For instance, if `phase_acc` has a width of 32-bits and we have an address width `ADDR_WIDTH` of 8 then bits `31..24` are used of the `phase_acc` for the `lut_addr`.  The `dds_sample` register is then set by indexing into the `sine_lut` register array with the `lut_addr`.

The register array `sine_lut` is implemented directly in the `dds.v` source.  Here it is shown for a 256-element table with 8-bit values.  

```verilog
initial begin
    sine_lut[  0] = 8'h80;
    sine_lut[  1] = 8'h83;
    sine_lut[  2] = 8'h86;
...
    sine_lut[255] = 8'h7C;
    end
```

The logic is that as the `phase_step` value increases that on each clk tick we step through the LUT faster increasing the frequency of the waveform.  

#### SPI master and DAC control  

The `ad5541_spi.v` module implements a SPI master with the following inputs and outputs:

```verilog
    input wire clk,
    input wire rst,
    input wire start,
    input wire [15:0] data_in,
    
    output reg busy,
    
    // AD5541A pins
    output reg sync_n,
    output reg sclk,
    output reg sdin
```

The initialization looks like:

```verilog
ad5541_spi #(
        .CLK_DIV(2)    // 20 MHZ equivalent 
    ) spi_dac (
        .clk(S_AXI_ACLK),
        .rst(ctrl_reset),
        .start(start_spi),
        .data_in(dac_data_reg),
        .busy(spi_busy),
        
        .sync_n(sync_n),
        .sclk(sclk),
        .sdin(sdin)
    );
```

A wire input for the `clk` as `S_AXI_CLK`. 

A wire input for `rst` linked to `ctrl_reset`.  The `ctrl_reset` is assigned from the 2nd bit of the `slv_reg0` which is the AXI lite write register.

The `start_spi`, `dac_data_reg`, and `spi_busy` are connected to registers in the top level.  

The AD5541A pins are connected to nets defined in our constraint file `Zybo_Master.xdc`.  

```
##Pmod Header JC
##IO_L10N_T1_34
set_property PACKAGE_PIN W15 [get_ports sdin]
set_property IOSTANDARD LVCMOS33 [get_ports sdin]

##IO_L10P_T1_34
set_property PACKAGE_PIN V15 [get_ports sync_n]
set_property IOSTANDARD LVCMOS33 [get_ports sync_n]

##IO_L1N_T0_34
set_property PACKAGE_PIN T10 [get_ports sclk]
set_property IOSTANDARD LVCMOS33 [get_ports sclk]

##IO_L1P_T0_34
set_property PACKAGE_PIN T11 [get_ports ldac_n]
set_property IOSTANDARD LVCMOS33 [get_ports ldac_n]
```

The module implements a state machine with states for `IDLE`, `SHIFT` and `DONE`.  When in the `SHIFT` state a register for `clk_div` counts the number of `clk` ticks until the parameter `CLK_DIV` counts.  When this condition happens there is logic to use a shift register to copy out the bit of the 16-bit `shift_reg` and to toggle the `sclk`.  Upon sending 16-bits worth of data, as measured by the `bit_cnt` register then the state machine moves to the `DONE` state.  In the `DONE` state the busy signal is released (0) and the state is moved back to `IDLE` allowing the next transaction to occur.  

#### top

The main logic is implemented in the `dds_dac_core_v1_0_S00_AXI.v` file.  There are two things going on here.

The first, is that we update the `dac_data_reg` by zero-padding the lower bits with the current `dds_sample` wire.  The `dds_sample` is updated with the 8-bit DDS wave value by the `dds` module described above.  We have to zero pad the value because our DAC operates at 16-bit width and the DDS does 8-bit output.  We also have a check if `reset` is enabled by user then we clear the `dac_data_reg` register.  

```verilog
always @(posedge S_AXI_ACLK) begin
    if (ctrl_reset) begin
        dac_data_reg <= 16'd0;
    end else if (ctrl_enable && !spi_busy && sample_div == SAMPLE_DIV_MAX) begin
        dac_data_reg <= { dds_sample, 8'b0 };
    end
end
```

The other thing this code does is start SPI transaction with the `ad5541_spi` module.  We keep a counter `sample_div` and wait till it gets to `SAMPLE_DIV_MAX` ticks before we start a SPI transaction.  To start a SPI transaction we set `start_spi` register high.  

We also force `ldac_n` output pin (constraint) to an always zero (0) state.  This forces the DAC to always be responsive to updates in the clocked digital value we send and simplifies need to update it in our loop.  

### Petalinux UIO

The Vivado image will expose the AXI Lite Component used for the `Enable`, `Reset`, and `phase_step` at a memory address.  The conventional way is just memory map `mmap` the physical address and get a mapped virtual pointer back.  

A cleaner (and more modern) way is to use Linux UIO (userspace I/O).  Main benefit of UIO you can confine a specific region (physical address and size) to prevent unintended writes and also have a centralized `/de/uioX` devices for each region your system addresses.  

Below, make the modification to the `project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi`.  The file should look like:

```
/include/ "system-conf.dtsi"
/ {
	chosen {
		bootargs = "console=ttyPS0,115200 earlyprintk uio_pdrv_genirq.of_id=generic-uio";
		stdout-path = "serial0:115200n8";
	};

    amba_pl {
		#address-cells = <0x1>;
		#size-cells = <0x1>;
		ranges;

		My_PWM_Core@43c00000 {
			compatible = "generic-uio";
			reg = <0x43c00000 0x10000>;
			interrupts = <0 0 0>;
		};
	};
};
```

#### Petalinux Build & Package

If you are starting from Vivado, export the HDF/BIT files.  Follow the directions below to create bootable files.

1. Create a new directory.  Run this command to import the hardware configuration from the Vivado project.  

```shell
mkdir -p dds_example && cd dds_example
petalinux-create -t project --template zynq --name project
cp -r /path/to/vivado/files/* ./xsa
cd project
petalinux-config --get-hw-description=../xsa/
```

1. Run the `petalinux-config -c kernel`

Ensure that `Device Drivers -> Userspace I/O drivers` has a selection for `Userspace I/O platform driver` and `Userspace I/O interrupt support` and that both are enabled.

1. Run the `petalinux-config -c rootfs`

Ensure that `python3`, `python3-mmap` and `vim` are enabled. 

1. Build and Package the `BOOT.BIN` and `image.ub `files from the previous build.  Copy the files to an SD card with the partition formatted as FAT32 type.

```shell
petalinux-build
petalinux-package --boot --fsbl images/linux/zynq_fsbl.elf --fpga ../xsa/design_1_wrapper.bit --u-boot --force
```

### Testing

1. Set your Zynq board to boot from SD card with the boot jumper.  Power the board on.  Connect a Putty or serial terminal session to the USB UART for the board.  Login to the petalinux board using default credentials `root:root`.

1. Run the example `dds=control.py` script.  This will prompt with an input to select the `phase_step` parameter and program it using the memory mapped `/dev/uio0` device for register memory on the Vivado design.

Write the value `1000000` and confirm that the sine wave observed changes frequency.

![Salae Logic Analyzer Screenshot](/assets/images/fpga-dds/logic_analyzer_screenshot.png)

1. You can also run the Vivado ILA and examine the registers previously exposed. 

With the board connected with USB.  Click the `Open Hardware Manager` option.  Click `Open target` and select `Auto Connect`.

If the FPGA is not programmed then right click your target and click `Program Device`.

Add the nets you previously defined in the `Debugging` section by clicking the `+` button in the waveform pane.

![ILA waveform add](/assets/images/fpga-dds/ila_waveform_add.png)

Select all the nets from the list and click OK.

![ILA Add Waveform Nets](/assets/images/fpga-dds/ila_debug_net_names.png)

You can setup trigger to capture but as the test above with the python `dds-control.py` specifies a phase step and configures the `ENABLE` and `RESET` you can just run an immediate trigger.

![ILA immediate trigger](/assets/images/fpga-dds/ila_immediate_trigger.png)


You can see from the ILA capture window below.  The `CTRL_ENABLE` is 1 (enabled).  The `CTRL_RESET` is 0 for not enabled.  

You can see the 32-bit `phase_acc` value advancing.  You also the `lut_addr` which is formed by the upper bit selection fo the `phase_acc` following it.  You also see the `dds_sample` value change as the `lut_addr` changes since it's the contents of the LUT at the `lut_addr` index.

You can also see the internal SPI master debug nets from `ad5541_spi.v` showing `bit_cnt`, `shift_reg`, `clk_div`, and the internal state machine `state`.

![ILA capture analysis](/assets/images/fpga-dds/ila_capture_analysis.png)

