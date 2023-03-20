
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

![image](https://user-images.githubusercontent.com/83432874/226153375-6c55cb67-bff6-460b-ab47-fdc112b4f77a.png)

And data being written when miss occurs in l2 cache after using ```mem_write 0x50000000 0x12345678```:

![image](https://user-images.githubusercontent.com/83432874/226153657-0a1a2a87-b5b8-4ee2-a854-621efd92bbce.png)

The wdata_payload_data signal is set with 0x12345678 in its least significant part. I noted here that the wdata_payload_we signal is set to all ones while the data value is first read from the l2 cache and written to the dram with the data being placed in the correct spot. 

Here is the behavior of a read from address 0x0 (after using ```mem_read 0x40000000 0x12345678 128```):

![image](https://user-images.githubusercontent.com/83432874/226159445-9b488a5d-742f-4732-a721-f84efc266c6d.png)

Where the data where r_data_valid is high has 0x12345678 appearing only once in the least significant part of rdata_payload_data.

This is the repeated output with the vexriscv cpu.

```
litex> mem_write 0x40000000 0x12345678       

litex> mem_write 0x50000000 0x12345678 

litex> mem_read 0x40000000 128
Memory dump:
0x40000000  78 56 34 12 02 00 30 ff 21 00 18 60 03 00 2c d6  xV4...0.!..`..,.
0x40000010  03 00 36 d8 01 00 1b c0 01 80 2d b6 02 c0 36 80  ..6.......-...6.
0x40000020  02 60 9b 6d 03 b0 ed 4c 03 d8 56 db 01 6c ab d2  .`.m...L..V..l..
0x40000030  01 b6 f5 b6 02 db 5a b6 02 6d ad 6d c3 b6 f6 44  ......Z..m.m...D
0x40000040  ff 5b 5b db b1 ad ad ff 62 d6 f6 b6 6e 6b 5b 4d  .[[.....b...nk[M
0x40000050  9b b5 ad 6d d8 da f6 db b7 6d 7b 5b b6 b6 bd b6  ...m.....m{[....
0x40000060  6c db de 16 ae 6d 4f 2d 5b b6 a7 45 68 db f3 a2  l....mO-[..Eh...
0x40000070  97 ed 79 51 da f6 bc a2 b4 7b 5e 14 b5 3d 0f 08  ..yQ.....{^..=..
```

Currently I think the wdata_payload_we signal isn't being used correctly in LiteDRAM for this large of a data width and address width.



