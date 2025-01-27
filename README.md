# PS5-offsets.sh
Kstuff 
# Remove old temporary files
rm -f payload.elf payload.bin r0run.o prosper0gdb.o

# Navigate to the library folder and build components
cd ../lib
make  # Build the library

# Assemble and compile necessary components
yasm -f elf64 crt.asm           # Assemble boot file
yasm -f elf64 rfork.asm         # Assemble fork commands
gcc -c -isystem ../freebsd-headers -nostdinc -fno-stack-protector dl.c -o dl.o -fPIE -ffreestanding  # Compile dynamic library

# Create system calls for PS5
python3 syscalls.py > syscalls.asm
yasm -f elf64 syscalls.asm
ld -r crt.o rfork.o dl.o syscalls.o -o lib.a

python3 syscalls-ps5.py > syscalls-ps5.asm
yasm -f elf64 syscalls-ps5.asm
ld -r crt.o dl.o syscalls-ps5.o -o lib-ps5.a

# Build ELF executable files
yasm -f elf64 crt-elf.asm
gcc -c -isystem ../freebsd-headers -nostdinc -fno-stack-protector -O3 crt-elf-c.c -o crt-elf-c.o -fPIE -ffreestanding
ld -r crt-elf.o crt-elf-c.o dl.o syscalls.o -o lib-elf.a

# Return to the parent directory
cd ..

# Compile the main program
yasm -f elf64 -g dwarf2 r0run.asm -o r0run.o
gcc -O0 -g -isystem ../freebsd-headers -nostdinc -nostdlib -fno-stack-protector -r -Wl,--unique='*' -ffunction-sections -fdata-sections \
    -DMEMRW_FALLBACK -DNO_BUILTIN_OFFSETS r0gdb.c r0run.o offsets.c -o prosper0gdb.o -fPIE -ffreestanding -fno-unwind-tables -fno-asynchronous-unwind-tables

gcc -O0 -g -isystem ../freebsd-headers -nostdinc -nostdlib -fno-stack-protector -static ../lib/lib-elf.a -DMEMRW_FALLBACK -DNO_BUILTIN_OFFSETS \
    main.c prosper0gdb.o dbg.c -o payload.elf -fPIE -ffreestanding -Wl,-no-pie -Wl,-zmax-page-size=16384 -Wl,-zcommon-page-size=16384

# Convert ELF to binary format
objcopy payload.elf --only-section .text --only-section .data --only-section .bss --only-section .rodata -O binary payload.bin

# Prepare custom ELF file
python3 ../lib/frankenelf.py payload.bin

# Connect to PS5
echo "Connecting to PS5..."
# Include debugging and testing tools
echo "Connecting GDB..."
echo "warning: remote target does not support file transfer, attempting to access files from local filesystem."
echo "Dumping kernel data..."
