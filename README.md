This is the readme file

# APSP assignment: input data

Directory: `/scratch/apsp_assignment/`

| File | Size (bytes) | Contents |
|---|---|---|
| `apsp_input_N20000.bin` | 1,600,000,000 | Input weight matrix, N = 20000. **Use this file for the MPI assignment.** |
| `sample_input_N1000.bin` | 4,000,000 | Small input matrix, N = 1000, made by the same rules |
| `sample_output_N1000.bin` | 4,000,000 | Correct output for `sample_input_N1000.bin` |
| `SHA256SUMS` | small | Checksums of the three files above |
| `README.md` | small | This file |

Your programs read the input directly from this directory. You cannot write into
this directory. Keep all your own files (programs, job scripts, output files) in your
own directory `/scratch/<your login name>/`, which already exists.
`echo /scratch/$USER` prints its exact path.


## 1. What the matrix contains

- The graph has N vertices, numbered 0 to N-1.
- `W[i][j]` is the weight of the directed edge from vertex i to vertex j.
  `W[i][j]` and `W[j][i]` are unrelated.
- `W[i][i] = 0` for every i.
- `W[i][j] = 1000000000` (10^9) means there is no edge from i to j.
- Every other entry is an edge weight: an integer from 10 to 8000.
  There are no negative weights, so there are no negative cycles.

In `apsp_input_N20000.bin`, 227,994,014 of the 399,980,000 off-diagonal entries (57.0 %)
are edges. The other 43.0 % are `1000000000`.


## 2. File format

- The file is binary, not text. A text editor shows unreadable characters.
- Each entry is a 4-byte signed integer (`int32_t` in C), little-endian.
  Little-endian is the native byte order of the cluster, so reading the bytes
  into an `int32_t` array gives the correct values without any conversion.
- Entries are stored row by row:
  `W[0][0], W[0][1], ..., W[0][N-1], W[1][0], W[1][1], ..., W[N-1][N-1]`.
- There is no header, no separator and no padding.
  The file holds exactly N x N entries, so its size is N x N x 4 bytes.
- N is not stored in the file. N = 20000 for `apsp_input_N20000.bin` and N = 1000
  for the sample files.
- Entry `W[i][j]` starts at byte offset `(i x N + j) x 4`.
  Row i starts at byte offset `i x N x 4`.

To print the first 8 entries of row 0 from the shell:

```
od -A d -t d4 -N 32 /scratch/apsp_assignment/apsp_input_N20000.bin
```

To print the first 8 entries of row 3 (byte offset 3 x 20000 x 4 = 240000):

```
od -A d -t d4 -j 240000 -N 32 /scratch/apsp_assignment/apsp_input_N20000.bin
```

Each line of `od` output starts with a byte offset, followed by the values.


## 3. Output

The result is the N x N distance matrix D. It uses the same layout as the input:
4-byte little-endian signed integers, row by row, no header.

Entry `D[i][j]` of the output:

| Case | Value of `D[i][j]` |
|---|---|
| There is no path from i to j | `1000000000` |
| Otherwise | The length of a shortest path from i to j |

- `D[i][i] = 0` for every i.
- A shortest path has at most N-1 edges of weight at most 8000, so every shortest
  path length is less than 20000 x 8000 = 160,000,000. It can never be confused
  with `1000000000`.


## 4. Common mistakes

1. **32-bit offsets.** For N = 20000 the largest byte offset, 1,599,999,996, still fits
   in an `int`, but for any N above 23170 it does not. Compute byte offsets with a
   64-bit type: `(int64_t)i * N * 4`, `off_t`, or `MPI_Offset`.
2. **Memory.** The full matrix needs 1.6 GB as `int32_t`. A compute node has about
   125 GB of memory, shared by all processes on that node. If each of 32 processes
   on a node keeps a full copy, together they need 51 GB.
3. **The value 1000000000 is not a weight.** It means "no edge" (input) or "no path"
   (output), and it must appear unchanged in your output where there is no path.
   Keep `1000000000` as the "no path" value inside your program. Then
   `D[i][k] + D[k][j]` is at most 2,000,000,000, which fits in an `int32_t`, and a
   sum that contains `1000000000` is never smaller than an existing entry, so the
   usual update `if (D[i][k] + D[k][j] < D[i][j]) D[i][j] = D[i][k] + D[k][j];` is
   correct without extra checks. Do not use `INT_MAX` as "no path":
   `INT_MAX + 10` overflows and becomes negative.


## 5. Reading one row in C

```c
#define _FILE_OFFSET_BITS 64
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>

int main(void)
{
    const char *path = "/scratch/apsp_assignment/apsp_input_N20000.bin";
    const int64_t N = 20000;
    const int64_t i = 3;                                  /* row to read */

    FILE *f = fopen(path, "rb");
    if (f == NULL) { perror(path); return 1; }

    int32_t *row = malloc(N * sizeof(int32_t));
    if (fseeko(f, (off_t)(i * N * 4), SEEK_SET) != 0) { perror("fseeko"); return 1; }
    if (fread(row, sizeof(int32_t), N, f) != (size_t)N) { fprintf(stderr, "short read\n"); return 1; }
    fclose(f);

    for (int j = 0; j < 8; j++)
        printf("W[%lld][%d] = %d\n", (long long)i, j, row[j]);

    free(row);
    return 0;
}
```

Compile with `gcc -O2 -o read_row read_row.c`.


## 6. Reading and writing with MPI-IO

Each process reads a contiguous block of rows with one collective call, and writes
its rows of the result to the output file in the same positions with one collective call.

```c
#include <mpi.h>
#include <stdint.h>
#include <stdlib.h>

#define N 20000

int main(int argc, char **argv)
{
    MPI_Init(&argc, &argv);
    int rank, size;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    /* This process owns rows first .. first + count - 1. */
    int base = N / size, extra = N % size;
    int count = base + (rank < extra ? 1 : 0);
    int first = rank * base + (rank < extra ? rank : extra);

    /* One matrix row as an MPI datatype: the calls below transfer count rows. */
    MPI_Datatype row;
    MPI_Type_contiguous(N, MPI_INT32_T, &row);
    MPI_Type_commit(&row);

    int32_t *local = malloc((size_t)count * N * sizeof(int32_t));
    MPI_Offset offset = (MPI_Offset)first * N * 4;
    MPI_File fh;

    MPI_File_open(MPI_COMM_WORLD,
                  "/scratch/apsp_assignment/apsp_input_N20000.bin",
                  MPI_MODE_RDONLY, MPI_INFO_NULL, &fh);
    MPI_File_read_at_all(fh, offset, local, count, row, MPI_STATUS_IGNORE);
    MPI_File_close(&fh);

    /* ... compute the shortest distances for the local rows here ... */

    MPI_File_open(MPI_COMM_WORLD, "my_output_N20000.bin",
                  MPI_MODE_CREATE | MPI_MODE_WRONLY, MPI_INFO_NULL, &fh);
    MPI_File_set_size(fh, (MPI_Offset)N * N * 4);   /* removes data left over from an older, larger file */
    MPI_File_write_at_all(fh, offset, local, count, row, MPI_STATUS_IGNORE);
    MPI_File_close(&fh);

    MPI_Type_free(&row);
    free(local);
    MPI_Finalize();
    return 0;
}
```

Compile with `mpicc -O2 -o apsp_io apsp_io.c`.
Run it with `srun --mpi=pmix ./apsp_io` inside a Slurm job, or with `mpirun ./apsp_io`.
A plain `srun ./apsp_io` without `--mpi=pmix` fails in `MPI_Init` on this cluster.

**Write with few, large calls.** `/scratch` is a network file system, and every
`MPI_File_write_at` call takes a file lock. Writing the 1000 x 1000 sample with one
call per row segment of a 2D block took 135 seconds with 64 processes; one collective
call per process took 3 seconds. For a 2D block, describe the block with
`MPI_Type_create_subarray`, set it as the file view with `MPI_File_set_view`, and write
it with one `MPI_File_write_all` call.


## 7. Looking at values from Python

```python
import numpy as np
N = 20000
W = np.memmap("/scratch/apsp_assignment/apsp_input_N20000.bin",
              dtype="<i4", mode="r", shape=(N, N))
print(W[3, :8])        # first 8 entries of row 3
```

`np.memmap` reads only the entries you access. Run this with `/usr/bin/python3` on the
login node; numpy is installed there, but not on every compute node.


## 8. Testing your program on the sample

`sample_input_N1000.bin` follows the same rules as the main file (missing edges,
weights from 10 to 8000, some pairs with no path). `sample_output_N1000.bin` is its correct output.

Run your program on the sample with N = 1000, then compare:

```
cmp /scratch/apsp_assignment/sample_output_N1000.bin my_output_N1000.bin
```

- No output from `cmp` means the files are identical.
- Otherwise `cmp` reports the first differing byte number b (counted from 1).
  The differing entry is e = (b - 1) / 4 (integer division),
  in row i = e / 1000 and column j = e % 1000.


## 9. Checking the files

`SHA256SUMS` holds the checksums of the three data files. To check the files:

```
cd /scratch/apsp_assignment && sha256sum -c SHA256SUMS
```

Each line must end with `OK`.
