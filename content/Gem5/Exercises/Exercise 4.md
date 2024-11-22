## Purpose
The purpose of this exercise will be to address some of the issues with the prior exercise (3). One issue is that the execution of the row major function in `main()` causes a couple of issues. The primary issue is that it makes it difficult to analyze the stats. Another issue is that it causes the row major function to have potential access to precached data before the execution of that "thread" is even invoked. The solution to this problem is to add a third core so that each thread gets executed independently of `main()`.

Another issue is that the simplicity in the summing algorithm makes it difficult to draw a clear distinction between the row major and column major threads when it comes to the benchmarking speeds. The solution here will be to randomize the row/column that the threads index for.

Another issue is associating memory transfers to specific instructions. The solution will be to add a custom debug message that prints the program counter (PC) within the packet of the top level (L1) memory request.

A final issue is that the use of a shared pointer to the same matrix's memory address means that the threads aren't fighting for space when the L2 cache is shared between them. The solution here will be to have two copies of the same matrix, each with their own location in DRAM, so that the threads are forced to treat each other's matrix values as irrelevant and evict it.
## Program
### Source Code
The first key difference between `sum-4.c` and `sum-3.c` (from the last exercise) is that this time the row and column indexes, within the summing threads, are shuffled. The second key difference is that both functions are assigned their own respective thread rather than one of them being executed in `main()`. The third distinction is that, despite both matrices being identical in data, the threads are not accessing the same matrix.
### sum-4.c
```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <stdbool.h>
#include <sys/time.h>

#define NUM_ROWS 50
#define NUM_COLUMNS 50

static uint8_t row_indexes[NUM_ROWS] = {0};
static uint8_t column_indexes[NUM_COLUMNS] = {0};

static void shuffle(uint8_t arr[], uint8_t arr_len) {
    // Seed the random number generator
    srand(time(NULL));
    
    for (uint8_t i = arr_len - 1; i > 0; i--) {
        // Pick a random index from 0 to i
        uint8_t j = rand() % (i + 1);

        // Swap arr[i] with the element at random index
        uint8_t temp = arr[i];
        arr[i] = arr[j];
        arr[j] = temp;
    }
}

static void generate_indexes(uint8_t *indexes) {
    for (uint8_t i = 0; i < NUM_ROWS; i++) {
        indexes[i] = i;
    }
}

struct matrix_info {
    uint32_t **matrix;
};

void *sum_by_row_major(void *threadid) {
    struct timeval t_start, t_stop, t_result;
    gettimeofday(&t_start, NULL);
    struct matrix_info *info = (struct matrix_info *) threadid;
    printf("Row major thread started\n");
    uint32_t sum = 0;
    for (int i = 0; i < NUM_ROWS; i++) {
        uint8_t row = row_indexes[i];
        for (int j = 0; j < NUM_COLUMNS; j++) {
            sum += info->matrix[row][j];
        }
    }
    gettimeofday(&t_stop, NULL);
    timersub(&t_stop, &t_start, &t_result);
    printf("Row major thread finished in %u microseconds\nSum: %u\n", (long int)t_result.tv_usec, sum);
}

void *sum_by_column_major(void *threadid) {
    struct timeval t_start, t_stop, t_result;
    gettimeofday(&t_start, NULL);
    struct matrix_info *info = (struct matrix_info *) threadid;
    printf("Column major thread started\n");
    uint32_t sum = 0;
    for (int j = 0; j < NUM_COLUMNS; j++) {
        uint8_t column = column_indexes[j];
        for (int i = 0; i < NUM_ROWS; i++) {
            sum += info->matrix[i][column];
        }
    }
    gettimeofday(&t_stop, NULL);
    timersub(&t_stop, &t_start, &t_result);
    printf("Column major thread finished in %u microseconds\nSum: %u\n", (long int)t_result.tv_usec, sum);
}

int main() {
    generate_indexes(row_indexes);
    generate_indexes(column_indexes);
    shuffle(row_indexes, NUM_ROWS);
    shuffle(column_indexes, NUM_COLUMNS);
    uint32_t **matrix1 = (uint32_t **)malloc(NUM_ROWS * sizeof(uint32_t *));
    uint32_t **matrix2 = (uint32_t **)malloc(NUM_ROWS * sizeof(uint32_t *));

    for (int i = 0; i < NUM_ROWS; i++) {
        matrix1[i] = (uint32_t *)malloc(NUM_COLUMNS * sizeof(uint32_t));
        matrix2[i] = (uint32_t *)malloc(NUM_COLUMNS * sizeof(uint32_t));
    }

    for (int i = 0; i < NUM_ROWS; i++) {
        for (int j = 0; j < NUM_COLUMNS; j++) {
            matrix1[i][j] = j;
            matrix2[i][j] = j;
        }
    }
    struct matrix_info info1;
    struct matrix_info info2;
    info1.matrix = matrix1;
    info2.matrix = matrix2;
    
    pthread_t column_major_thread;
    pthread_t row_major_thread;
    pthread_create(&column_major_thread, NULL, sum_by_column_major, (void *)&info1);
    pthread_create(&row_major_thread, NULL, sum_by_row_major, (void *)&info2);

    pthread_join(column_major_thread, NULL);
    pthread_join(row_major_thread, NULL);
    return 0;
}
```
### Compilation
As I was writing `sum-4.c`, I noticed an interesting issue. I had previously written and compiled the program in such a way that both threads were accessing the same matrix. This was its own issue as previously discussed. However, when I updated the program to fix this issue (by having a duplicate matrix), my VSCode git extension was showing that the compiled binary didn't change. This bug was actually a feature of the compiler optimizing my program, and effectively nullifying my changes, as I had forgotten to add the `O0` flag when compiling. By running the following Makefile script with compiler optimization disabled, my git extension showed that the `sum-4` binary had changed.
```makefile
../bin/x86/linux/sum-4: sum-4.c
	gcc -o ../bin/x86/linux/sum-4 -O0 sum-4.c -pthread
```

We can then run this Makefile using the `make` command
```bash
 >> make
```

## Simulation
### Custom Debug Message
We add the following `DPRINT` statement to the `access()` method of the `BaseCache` class in `base.cc`. This way, every cache level prints the PC of the associated read request. 

We call the `getPC()` method of the request member of the packet object to obtain the PC. Note the `if` statement conditional, this is to ensure that the simulation doesn't crash if the `access()` method is called with a packet request that doesn't contain a PC.
#### base.cc
```cpp
    if (pkt->req->hasPC()) {
        DPRINTF(Cache, "PC %x\n", pkt->req->getPC());
    }
```

### Configuration
The only difference between this exercise's configuration script and the one from exercise 3 is the system's number of cores.
```py
# Define number of cores
NUM_CORES = 3
```

### Execution
We can run the simulation with a shared L2 cache using the following command. We can also play with the L1 and L2 cache sizes.
```bash
 >> build/X86/gem5.opt configs/exercises/exercise-4/multi_core_system.py --l1d_size='2kB' --l2_size='8kB' --shared_l2
```

We then get the following output which shows us that our program was successfully executed.
```txt
Column major thread started
Row major thread started
Row major thread finished in 81 microseconds
Sum: 61250
Column major thread finished in 152 microseconds
Sum: 61250
Exiting @ tick 704537091 because exiting with last active thread context
```

To generate and log debug messages in a `typescript` file, we run the following commands.
```bash
 >> script
 >> build/X86/gem5.opt --debug-flags=DRAM,Exec,Cache configs/exercises/exercise-4/multi_core_system.py --l1d_size='2kB' --l2_size='8kB' --shared_l2
 >> exit
```

### Analysis
Since the column major thread is slower, we can probably guess that it experienced more cache misses than the row major thread. For this reason, we should look at the column major thread's instructions to find the one that causes the cache misses. We can use the following command to compile the assembly program with assisting comments.
```bash
 >> gcc -S -fverbose-asm -O0 sum-4.c -o sum-4-nice.asm
```

The fourth instruction in this snippet shows the exact value being read from the matrix. We're interested in the PC of this instruction.
#### sum-4-nice.asm
```asm
# sum-4.c:65:             sum += info->matrix[i][column];
	movzbl	-25(%rbp), %edx	# column, _6
	salq	$2, %rdx	#, _7
	addq	%rdx, %rax	# _7, _8
	movl	(%rax), %eax	# *_8, _9
```

Next, we run objdump on our originally compiled binary to generate the assembly instructions with their respective PC addresses.
```bash
 >> objdump -d sum-4 > sum-4-with-pcs
```

We then find the PC address that we're looking for, 0x140F.
#### sum-4-with-pcs
```
    1404:	0f b6 55 e7          	movzbl -0x19(%rbp),%edx
    1408:	48 c1 e2 02          	shl    $0x2,%rdx
    140c:	48 01 d0             	add    %rdx,%rax
    140f:	8b 00                	mov    (%rax),%eax
```

We can then trace the debug messages for this specific instruction and can clearly see the memory transfers between the dcache, l2cache, and DRAM as they lead up to the execution of the instruction.
```
594071334: system.cpu1.dcache: access for ReadReq [9459c:9459f] miss
594071334: system.cpu1.dcache: PC 140f
594072000: system.cpu1.dcache: sendMSHRQueuePacket: MSHR ReadReq [9459c:9459f]
594072000: system.cpu1.dcache: createMissPacket: created ReadSharedReq [94580:945bf] from ReadReq [9459c:9459f]

594072000: system.l2cache: access for ReadSharedReq [94580:945bf] miss
594072000: system.l2cache: PC 140f
594078993: system.l2cache: sendMSHRQueuePacket: MSHR ReadSharedReq [94580:945bf]
594078993: system.l2cache: createMissPacket: created ReadSharedReq [94580:945bf] from ReadSharedReq [94580:945bf]

594078993: system.mem_ctrl.dram: Address: 0x94580 Rank 0 Bank 10 Row 2
594078993: system.mem_ctrl.dram: Timing access to addr 0x94580, rank/bank/row 0 10 2
594078993: system.mem_ctrl.dram: Schedule RD/WR burst at tick 594078993
594081167: system.mem_ctrl.dram: number of read entries for rank 0 is 1
594096485: system.mem_ctrl.dram: number of read entries for rank 0 is 0

594071334: system.cpu1: T0 : 0x140f @sum_by_column_major+130    : mov	eax, DS:[rax]         
594071334: system.cpu1: T0 : 0x140f @sum_by_column_major+130. 0 :   MOV_R_M : ld   eax, DS:[rax] : MemRead :  D=0x0000000000000017 A=0x959c
```

The debug messages show us that cpu1 executes the column major thread and cpu2 executes the row major thread so these are these are the cores we're interested in when digging through `stats.txt`.

The following table gives us the stats for 4 different configurations of the simulation. As we would expect, using an L2 Cache consistently slows down execution. Additionally, a l1d and l2 cache size result in less cache misses and faster execution. 

Interestingly, switching to a shared l2 cache for the smaller cache configuration results in a 0.6% increase in cpi while only resulting in a 0.18% increase for the larger cache configuration. 

|                                             | l1dcache: 1kB<br>private l2cache: 4kB | l1dcache: 1kB<br>shared l2cache: 4kB | l1dcache: 2kB<br>private l2cache: 8kB | l1dcache: 2kB<br>shared l2cache: 8kB |
| ------------------------------------------- | ------------------------------------- | ------------------------------------ | ------------------------------------- | ------------------------------------ |
| row thread cycles                           | 2176699                               | 2162818                              | 1884464                               | 1881029                              |
| row thread instructions                     | 41290                                 | 41276                                | 41276                                 | 41276                                |
| row thread cpi                              | 52.717341                             | 52.398924                            | 45.655199                             | 45.571979                            |
| row thread l1dcache overall misses          | 2493                                  | 2490                                 | 623                                   | 619                                  |
| row thread private l2 cache overall misses  | 647                                   | N/A                                  | 619                                   | N/A                                  |
| column thread cycles                        | 2457633                               | 2463585                              | 2049265                               | 2074030                              |
| column thread instructions                  | 43354                                 | 43354                                | 43354                                 | 43354                                |
| column thread cpi                           | 56.687572                             | 56.824860                            | 47.268187                             | 47.839415                            |
| column thread l1dcache total misses         | 4507                                  | 4502                                 | 3272                                  | 3268                                 |
| column thread private l2 cache total misses | 2523                                  | N/A                                  | 1506                                  | N/A                                  |
| shared l2 cache total misses                | N/A                                   | 8571                                 | N/A                                   | 6438                                 |



