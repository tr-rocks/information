
# Write to 64-bit data-width with vexriscv_smp not working

I am currently using the antmicro data center dram tester board, and am trying to get it working with the vexriscv_smp cpu. I was able to run it with the vexriscv successfully. First I ran the command ```python litex-boards/litex_boards/targets/antmicro_datacenter_ddr4_test_board.py --load --build --cpu-type vexriscv_smp```, and then ```litex_server --uart --uart-port /dev/ttyUSB2```. Then in a separate terminal, I ran ```litex_term crossover```, and as the DRAM is initialized, it fails due to data errors. I also noticed strange behavior when writing a value to an address, in this case, the value 0x12345678 to 0x40000000:

```
litex> mem_write 0x40000000 0x12345678

litex> mem_read 0x40000000 128   
Memory dump:
0x40000000  78 56 34 12 78 56 34 12 78 56 34 12 78 56 34 12  xV4.xV4.xV4.xV4.
0x40000010  78 56 34 12 78 56 34 12 78 56 34 12 78 56 34 12  xV4.xV4.xV4.xV4.
0x40000020  78 56 34 12 78 56 34 12 78 56 34 12 78 56 34 12  xV4.xV4.xV4.xV4.
0x40000030  78 56 34 12 78 56 34 12 78 56 34 12 78 56 34 12  xV4.xV4.xV4.xV4.
0x40000040  b5 3d 0f 8a b5 3d 0f 8a b5 3d 0f 8a b5 3d 0f 8a  .=...=...=...=..
0x40000050  b5 3d 0f 8a b5 3d 0f 8a b5 3d 0f 8a b5 3d 0f 8a  .=...=...=...=..
0x40000060  b5 3d 0f 8a b5 3d 0f 8a b5 3d 0f 8a b5 3d 0f 8a  .=...=...=...=..
0x40000070  b5 3d 0f 8a b5 3d 0f 8a b5 3d 0f 8a b5 3d 0f 8a  .=...=...=...=..

litex> 
```
I noted the data is repeated over and over again up to address 0x40000040. Trying other values and addresses, I found this behavior of repeated data was the same writing to every address within this 0x40 space. (For example, writing 0xa5a5a5a5 to 0x40000030 would change all the values from 0x40000000 to 0x40000040 to 0xa5a5a5a5.)

Having access also to an alveo u280 board, I tried using the dram with the vexriscv and vexriscv_smp cpus, and recieved the exact same behavior, in that the implementation with the vexriscv worked fine while initialization failed with the vexriscv_smp with similar reading/writing behavior. Therefore, I know this isn't a board-specific problem for the antmicro data board.

At this point, I started using Litescope to look at the native protocol. The vexriscv_smp cpu has two busses with direct connections to LiteDRAM, stored in a list ```memory_buses```, and using litescope with the port declared on line 1527 of litex>soc>integration>soc.py connected to the dbus (2nd element of ```memory_buses```) and repeating the write, I got the following:

![image](https://user-images.githubusercontent.com/83432874/226158591-39436a0f-0fb7-4b6d-b6f8-2a480c19de56.png)

I noted that the data is 0x12345678 repeated every 0x4 address space in wdata_payload_data, and the wdata_payload_we signal is set with ones at the corresponding address.

Repeating the read, I got the following:

![image](https://user-images.githubusercontent.com/83432874/226158774-1b39ca60-c802-4553-97f1-ff790eae8a0e.png)

The value of the data, when rdata_valid goes high, is 0x12345678 repeated 16 times, when the data 0x12345678 should only appear once at the least significant part (at address 0x0).

As I mentioned before, I was able to run litex successfully with the vexriscv cpu, and I noticed different reading/writing behavior when analyzing the ports with litescope. Instead of two memory buses with direct connections to LiteDRAM, the vexriscv is actively using the port declared on line 1596 of the same soc.py and includes an l2 cache connected between two wishbone protocol interfaces before the litedram port. As the info is first read into the l2 cache when writing memory, I first wrote the value 0x12345678 to the address 0x40000000 and then far away to get the actual writing behavior to the dram: 

Data being read after using ```mem_write 0x40000000 0x12345678```:

![image](https://user-images.githubusercontent.com/83432874/226442130-f8898b49-ba91-49b8-b9c8-533276f7faa7.png)

And data being written when miss occurs in l2 cache after using ```mem_write 0x50000000 0x12345678```:

![image](https://user-images.githubusercontent.com/83432874/226442863-8dc6741d-f956-4051-b106-57e5f5551f2b.png)

The wdata_payload_data signal is set with 0x12345678 in its least significant part. I noted here that the wdata_payload_we signal is set to all ones while the data value is first read from the l2 cache and written to the dram with the data being placed in the correct spot. 

Here is the behavior of a read from address 0x0 (after using ```mem_read 0x40000000 0x12345678 128```):

![image](https://user-images.githubusercontent.com/83432874/226443811-ca5e41b9-9d28-4003-ba53-862cf7c3dd3c.png)

Where the data where r_data_valid is high has 0x12345678 appearing only once in the least significant part of rdata_payload_data.

This is the repeated output with the vexriscv cpu.

```
litex> mem_write 0x40000000 0x12345678       

litex> mem_write 0x50000000 0x12345678 

litex> mem_read 0x40000000 128
Memory dump:
0x40000000  78 56 34 12 ff ff ff ff ff ff ff ff ff ff ff ff  xV4.............
0x40000010  ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff  ................
0x40000020  ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff  ................
0x40000030  ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff  ................
0x40000040  ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff  ................
0x40000050  ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff  ................
0x40000060  ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff  ................
0x40000070  ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff  ................

litex> 
```

Currently I think the wdata_payload_we signal isn't being used correctly in LiteDRAM for this large of a data width and address width.



