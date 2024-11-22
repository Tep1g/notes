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