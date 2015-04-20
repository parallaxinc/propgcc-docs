---
layout: doc
title: Linker Scripts
---

This article shows how to use linker scripts that are included in the
PropGCC release to link xmm and xmmc programs to run from external RAM
or flash/eeprom.

Details
=======

Included linker scripts:

<table>
<thead>
<tr class="header">
<th align="left">-mxmm</th>
<th align="left">-T xmm.ld</th>
<th align="left">code in external flash/eeprom, data in external ram</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="left">-mxmm</td>
<td align="left">-T xmm_ram.ld</td>
<td align="left">code and data in external ram</td>
</tr>
<tr class="even">
<td align="left">-mxmmc</td>
<td align="left">-T xmmc.ld</td>
<td align="left">code in external flash/eeprom, data in hub memory</td>
</tr>
<tr class="odd">
<td align="left">-mxmmc</td>
<td align="left">-T xmmc_ram.ld</td>
<td align="left">code in external ram, data in hub memory</td>
</tr>
</tbody>
</table>

Note that the stack and locals are always in hub memory in all of the
memory models.

You should also make sure that the -m option matches the linker script.
In other words, -mxmmc should always use either xmmc.ld or xmmc\_ram.ld
and -xmm should always use xmm.ld or xmm\_ram.ld.

Example of how to link an xmmc program to run from external ram:

    propeller-elf-gcc -Os -o fibo_xmmc_ram.elf -mxmmc -T xmmc_ram.ld fibo.c
