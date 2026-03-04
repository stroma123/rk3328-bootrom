# Rockchip RK3328 BootROM

+ `rom.txt` - dump of the RK3328 BootROM, see https://github.com/trebisky/Rockchip/tree/master/bare_dump for details and all the credits.
+ `bootrom.txt` - The dissasembled bootrom file with annotations.

`make` will generate:
+ `bootrom.bin` - binary blob from `rom.txt` dump.
+ `bootrom.elf` - ELF wrapped `bootrom.bin`.
+ `bootrom.dis` - disassembled `bootrom.elf`.
