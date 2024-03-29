
Write-up for alloc8: untethered bootrom exploit for iPhone 3GS

alloc8 brings freedom to millions of iPhone 3GS devices, forever, by exploiting a powerful vulnerability in function malloc in the bootrom. Both revisions of iPhone 3GS bootrom are vulnerable, but old bootrom is also vulnerable to 24Kpwn, which is faster than alloc8.


*** JAILBREAK TOOL ***
ipwndfu:
https://github.com/axi0mX/ipwndfu


*** AFFECTED DEVICES ***
iPhone 3GS (old bootrom)
iPhone 3GS (new bootrom)


*** VULNERABILITY ***

This is not a typical heap bug. This is a bug in implementation of the heap itself.

void *malloc(size_t size);

In C programming language, function malloc should return NULL if it is unable to allocate memory of the requested size. Caller should check if returned pointer is NULL and handle the error:

void *pointer = malloc(size);

if (pointer == NULL) {
	// handle error
} else {
	// pointer is valid, continue
}

In S5L8920 bootrom (and some very old versions of iBoot) function malloc is not implemented correctly. When it is unable to allocate memory, instead of NULL it returns a pointer to memory address 0x8. Callers check if returned pointer is NULL and then treat that pointer as valid.


*** EXPLOITATION ***

First, we must be able to allocate enough data on the heap to make it run out of memory. Apple seems to have done a good job in reducing the attack surface of the bootrom, and it is far from certain that this can be achieved.

There seems to be only a single way to fill up the heap during a normal boot. If we add additional IMG3 images to NOR, bootrom will parse all of them and allocate a 44-byte structure on the heap for each image. However, only the first image, LLB, needs to be parsed, because bootrom never uses any other image from NOR. Although this is not a vulnerability by itself, Apple changed this behavior later, in S5L8930 bootrom, and it no longer seems to be possible to fill up the heap in bootrom during normal boot.

Second, once the heap is full, we must be able to use the pointer(s) to memory address 0x8 for reading or writing in a way that gets us arbitrary code execution before any panic or fatal memory corruption occurs.

On ARMv7 processors, the exception vector table is located at memory address 0x0. The exception vector table contains critical instructions and data used for handling exceptions. Corrupting this exception vector table is a technique commonly used for exploits on ARMv7 processors. 

Although the exception vector table in bootrom comes from read-only memory, the exception vector table data is cached in L1 data cache, and it is possible to change behavior of the exception vector table by overwriting this data. Overwriting instructions has no effect, because instructions are cached separately in L1 instruction cache, and writes to memory are cached in L1 data cache, not L1 instruction cache.

This data contains pointers to exception handlers used by bootrom, and changing any of these pointers to address of our shellcode makes the processor jump to our shellcode when that exception occurs.

When bootrom is parsing IMG3 images from NOR, it reads 4096 bytes of data from NOR at a time into a temporary buffer. When there is a large number of IMG3 images in NOR, this temporary buffer is the first one which cannot be allocated on the heap, and it gets allocated at memory address 0x8. At that point, 4096 bytes of data from NOR, which we have full control over, gets copied to memory starting at memory address 0x8.

This gives us the ability to write arbitrary data over data in the exception vector table and additional data which is located after it. We will flash a copy of 4096 bytes of data from bootrom to NOR and override the pointer to data abort exception handler, effectively using this primitive to override 4 bytes in the exception vector table in memory and keep everything else the same.

Once reading from NOR is complete, bootrom attempts to free the temporary buffer at memory address 0x8. 8 bytes located immediately before allocated memory are used for heap metadata, but for this bad pointer the metadata is invalid. This leads to a bad memory access in function free, which triggers a data abort exception and the processor jumps to our shellcode.


*** SHELLCODE ***

First 52 bytes of NOR contain an IMG2 header with NOR metadata that should not be changed. Before bootrom starts parsing images in NOR, the first 512 bytes of NOR are copied to memory allocated on the heap, but only 52 bytes are actually used. The remaining 460 bytes are unused and can be safely used for shellcode. Memory address where this data gets allocated is always the same.


*** POST-EXPLOITATION ***

To clean up, shellcode returns from exception, sets a new stack top, restores the original pointer for data abort handler, and frees 44-byte structures which are occupying most of the heap, leaving only the one required for normal boot, LLB.

At this point, shellcode can continue booting an unsigned LLB image from NOR or go to pwned DFU Mode and boot an unsigned image sent over USB.


