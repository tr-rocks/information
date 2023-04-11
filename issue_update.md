It appears that the 'we' signal is working as expected for the four phases. There were no "dm" bits in the platform file for the dram, and I found out from the documentation that this RDIMM doesn't support byte-enabled writing. So, when the 'we' reached the phy, this signal was being ignored. 

The vexriscv_smp, by default, is using two litedram-native ports connected to an ibus and a dbus. (See litex>soc>cores>cpu>vexriscv_smp>core.py in the function ```add_memory_buses``` on line 454 for where these ports are added, and see litex>soc>integration>soc.py in the function ```add_sdram``` where ```self.cpu.buses``` are being used.) These busses are not connected to an l2 cache, and thus it uses the controller as if byte-enabled reading/writing were possible. 

On writing this and reviewing the code, I realized I overlooked an argument being used in core.py called ```with_wishbone_memory```. This in turn sets the argument ```VexRiscvSMP.wishbone_memory``` to true, and it is this argument which controls whether these LiteDRAMNative ports are being used or not. I have built this project again with the following command:

```python litex-boards/litex_boards/targets/antmicro_datacenter_ddr4_test_board.py --build --load --cpu-type vexriscv_smp --with-wishbone-memory```

And was able to successfully run this with no errors.

```
Switching SDRAM to hardware control.
Memtest at 0x40000000 (2.0MiB)...
  Write: 0x40000000-0x40200000 2.0MiB     
   Read: 0x40000000-0x40200000 2.0MiB     
Memtest OK
Memspeed at 0x40000000 (Sequential, 2.0MiB)...
  Write speed: 70.3MiB/s
   Read speed: 62.4MiB/s
```


