# Embedded Systems & Edge AI

Software and hardware designed around physical device limits: an FPGA peripheral with a bare-metal driver, and a quantized neural network running on a microcontroller camera board.

**Skills demonstrated:** VHDL, AXI4-Lite, MicroBlaze, Vivado and Vitis, bare-metal C, TensorFlow Lite int8 quantization, and OpenMV MicroPython.

| Project | Summary | Source |
|---|---|---|
| [FPGA_crc8_accelerator](FPGA_crc8_accelerator.md) | Custom VHDL CRC-8 peripheral on AXI4-Lite, integrated with a MicroBlaze soft processor on an Artix-7 board and validated against a software reference in bare-metal C. | [github.com/mattdisk/FPGA_crc8_accelerator](https://github.com/mattdisk/FPGA_crc8_accelerator) |
| [nicla-vision-waste-classifier](nicla-vision-waste-classifier.md) | Six-class waste image classifier quantized to int8 TensorFlow Lite and run live on an Arduino Nicla Vision under OpenMV MicroPython. | [github.com/mattdisk/nicla-vision-waste-classifier](https://github.com/mattdisk/nicla-vision-waste-classifier) |

[Back to portfolio overview](../README.md)
