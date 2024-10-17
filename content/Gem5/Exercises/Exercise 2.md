## Purpose
In this exercise, we'll be tracing memory transfers as they go between the CPU, cache, and DRAM. This will let us see how information is stored and received by the low level hardware. To accomplish this, we'll be utilizing the simulator's debugging functionality. We'll also be diving into the source code of the various modules to figure out which debug messages we're looking for.
## Program
### Source Code
Our program for this exercise will look similar to the one from exercise 1. The key difference with this program is in its memory management. Rather than storing our integer array on the stack, we'll be utilizing the heap to dynamically allocate memory for it during runtime.
#### sum-2.c
```c
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <string.h>

static void sum_arr(uint8_t *arr, uint8_t arr_len, uint8_t *sum) {
    for (int i = 0; i < arr_len; i++) {
        *sum += arr[i];
    }
}

static uint8_t *allocate_mem(void) {
    // Create pointer to and allocate heap memory
    uint8_t *heap_ptr = (uint8_t*)malloc(4 * sizeof(uint8_t));

    // Store values in heap
    heap_ptr[0] = 1;
    heap_ptr[1] = 200;
    heap_ptr[2] = 32;
    heap_ptr[3] = 3;

    return heap_ptr;
}

int main() {

    uint8_t *heap_ptr = allocate_mem();

    // Calculate sum from the values in the heap
    uint8_t sum;
    sum_arr(heap_ptr, 4, &sum);

    // Free memory
    free(heap_ptr);

    // Print sum
    printf("%d", sum);

    return 0;
}
```
### Compilation
The following command compiles `sum-2.c` into a binary file with compiler optimization disabled.
```bash
 >> gcc -fverbose-asm -O0 sum-2.c -o sum-2
```

Further, we can use the following `objdump` command to get the program memory addresses of the individual instructions. These will be important later.
```bash
 >> objdump -d sum-2 > sum-2.asm
```
## Simulation
### Configuration
We can copy most of the `simple_system.py` configuration script from exercise 1 for this simulation.
### Debug Messages
Gem5 supports optional debug messages, these messages can be enabled as flags in the same terminal command that runs the simulation.

Looking in the `doBurstAccess` method of the `DRAMInterface` class within the `dram_interface.cc` module, we see the following print statement.
#### dram_interface.cc
```c++
DPRINTF(DRAM, "Timing access to addr %#x, rank/bank/row %d %d %d\n",
		mem_pkt->addr, mem_pkt->rank, mem_pkt->bank, mem_pkt->row);
```
### Execution
Since the debug messages will output such an insane amount of text to the terminal, we'll need to use the following set of commands to log everything displayed in the terminal. This will create a roughly 35 MB `typescript` file.
```bash
 >> script
 >> build/X86/gem5.opt --debug-flags=DRAM,Exec,CacheVerbose configs/exercises/exercise-2/simple_system.py
 >> exit
```

#### CPU to Cache
We'll start with the compilation of lines 17-20 in `sum-2.c` where four unsigned 8 bit integers are inserted into the heap.
##### sum-2.c
```c
    heap_ptr[0] = 1;
    heap_ptr[1] = 200;
    heap_ptr[2] = 32;
    heap_ptr[3] = 3;
```

Going back to the earlier `sum-2.asm` that we generated with `objdump`, we can cross reference the following program memory addresses in the `typescript` log file: `11be:11df`.
##### sum-2.asm
```asm
00000000000011a4 <allocate_mem>:
...
    11be:	c6 00 01             	movb   $0x1,(%rax) # heap_ptr[0] = 1;
    11c1:	48 8b 45 f8          	mov    -0x8(%rbp),%rax
    11c5:	48 83 c0 01          	add    $0x1,%rax
    11c9:	c6 00 c8             	movb   $0xc8,(%rax) # heap_ptr[1] = 200;
    11cc:	48 8b 45 f8          	mov    -0x8(%rbp),%rax
    11d0:	48 83 c0 02          	add    $0x2,%rax
    11d4:	c6 00 20             	movb   $0x20,(%rax) # heap_ptr[2] = 32;
    11d7:	48 8b 45 f8          	mov    -0x8(%rbp),%rax
    11db:	48 83 c0 03          	add    $0x3,%rax
    11df:	c6 00 03             	movb   $0x3,(%rax) # heap_ptr[3] = 3;
```

Using `ctrl+f`, we can search for the program memory addresses and find the expected debug messages that map onto the earlier four lines of code. Let's break down these debug messages.
##### typescript
```
234857241: system.cpu.dcache: access for WriteReq [8c2a0:8c2a0] hit state: e (M) writable: 1 readable: 1 dirty: 1 prefetched: 0 | tag: 0x11 secure: 0 valid: 1 | set: 0x10a way: 0
234857241: system.cpu.dcache: satisfyRequest for WriteReq [8c2a0:8c2a0] (write)
234857241: system.cpu: T0 : 0x11be @allocate_mem+26. 1 :   MOV_M_I : st   t1b, DS:[rax] : MemWrite :  D=0x0000000000000001 A=0x52a0
...
234861237: system.cpu.dcache: access for WriteReq [8c2a1:8c2a1] hit state: e (M) writable: 1 readable: 1 dirty: 1 prefetched: 0 | tag: 0x11 secure: 0 valid: 1 | set: 0x10a way: 0
234861237: system.cpu.dcache: satisfyRequest for WriteReq [8c2a1:8c2a1] (write)
234861237: system.cpu: T0 : 0x11c9 @allocate_mem+37. 1 :   MOV_M_I : st   t1b, DS:[rax] : MemWrite :  D=0x00000000000000c8 A=0x52a1
...
234864567: system.cpu.dcache: access for WriteReq [8c2a2:8c2a2] hit state: e (M) writable: 1 readable: 1 dirty: 1 prefetched: 0 | tag: 0x11 secure: 0 valid: 1 | set: 0x10a way: 0
234864567: system.cpu.dcache: satisfyRequest for WriteReq [8c2a2:8c2a2] (write)
234864567: system.cpu: T0 : 0x11d4 @allocate_mem+48. 1 :   MOV_M_I : st   t1b, DS:[rax] : MemWrite :  D=0x0000000000000025 A=0x52a2
...
234869229: system.cpu.dcache: access for WriteReq [8c2a3:8c2a3] hit state: e (M) writable: 1 readable: 1 dirty: 1 prefetched: 0 | tag: 0x11 secure: 0 valid: 1 | set: 0x10a way: 0
234869229: system.cpu.dcache: satisfyRequest for WriteReq [8c2a3:8c2a3] (write)
234869229: system.cpu: T0 : 0x11df @allocate_mem+59. 1 :   MOV_M_I : st   t1b, DS:[rax] : MemWrite :  D=0x0000000000000003 A=0x52a3
```

For this first message, we see how it show some pretty cool information such as the cache instance (dcache or icache), the function (WriteReq or ReadReq), and the contents of the packet (memory address, hit state).
```
234857241: system.cpu.dcache: access for WriteReq [8c2a0:8c2a0] hit state: e (M) writable: 1 readable: 1 dirty: 1 prefetched: 0 | tag: 0x11 secure: 0 valid: 1 | set: 0x10a way: 0
```

Taking a look at the source code for the cache module, we find the `DPRINTF()` statement that produces this debug message. Additionally, a comment that conveniently describes how the `access()` method handles requests from the CPU side. The `pkt` variable is a pointer to a `Packet` object. Packets hold the actual data that get passed between the CPU, Cache, and DRAM. Also note the call to `pkt->print()`, this is a method call that prints the contents of the packet.
##### cache.cc
```c++
/////////////////////////////////////////////////////
//
// Access path: requests coming in from the CPU side
//
/////////////////////////////////////////////////////
bool
Cache::access(PacketPtr pkt, CacheBlk *&blk, Cycles &lat,
              PacketList &writebacks)
{
...
        DPRINTF(Cache, "%s for %s\n", __func__, pkt->print());
```

Additionally, the reference to `WriteReq` maps onto the gem5 memory system diagram of the cache.
![Alt Text](https://www.gem5.org/assets/img/gem5_MS_Fig7.PNG)

The next line is more brief and it notifies us that the prior write request is fulfilled.
```
234857241: system.cpu.dcache: satisfyRequest for WriteReq [8c2a0:8c2a0] (write)
```

Looking inside the `satisfyRequest()` method in the `Cache` class's parent class, `base.cc`, we see the `DPRINTF()` statement that produces this debug message.
##### base.cc
```c++
void
Cache::satisfyRequest(PacketPtr pkt, CacheBlk *blk,
                      bool deferred_response, bool pending_downgrade)
{
...
		DPRINTF(CacheVerbose, "%s for %s (write)\n", __func__, pkt->print());
```

Within the header file, we can look at the method declaration's docstring for more context. `satisfyRequest()` updates the cache block hit by the CPU's request packet.
##### base.hh
```c++
/**
     * Perform any necessary updates to the block and perform any data
     * exchange between the packet and the block. The flags of the
     * packet are also set accordingly.
     *
     * @param pkt Request packet from upstream that hit a block
     * @param blk Cache block that the packet hit
     * @param deferred_response Whether this request originally missed
     * @param pending_downgrade Whether the writable flag is to be removed
     */
    virtual void satisfyRequest(PacketPtr pkt, CacheBlk *blk,
                                bool deferred_response = false,
                                bool pending_downgrade = false);
```
