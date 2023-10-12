## Basic High Performance Linpack guide

Accessible here: [https://www.netlib.org/benchmark/hpl](https://www.netlib.org/benchmark/hpl/)

Download the tar file onto machine and uncompress:

```bash
wget https://www.netlib.org/benchmark/hpl/hpl-2.3.tar.gz
tar -xf hpl-2.3.tar.gz
```

Builds easily. Load compiler, MPI library, BLAS library.

- Use `module avail` to see available modules.
- Use `module load` to load one.

Make a `build` directory somewhere, could be in the uncompressed `hpl` directory.

```bash
mkdir build
pwd # gives the absolute path
./configure -prefix /absolute/path/to/build
make
make install
```

Your executable will be in the `bin` folder of the `build` dir. **It will be called `xhpl`.**

You need to add a `HPL.dat` file if it doesn’t already exist.

```bash
touch HPL.dat
```

Regardless of whether it does, **use this tool to generate one**: [https://www.advancedclustering.com/act_kb/tune-hpl-dat-file/](https://www.advancedclustering.com/act_kb/tune-hpl-dat-file/)

### Configuring HPL

Inside `HPL.dat`:

- `N` should be `> 100,000`
- `NB` should be `192`

General rule for `p` and `q`:

- `P * Q = number of ranks`
- Make sure they're as square as possible
- `q > p`

**Things to consider:**
To check whether its working, set `N` to something low (10,000 or less) and the run should be really fast (but probably not a good result).
Turn this into a slurm script and increase N to what the website suggests (and maybe even a little bit higher!)
You can also try changing out compiler/mpi/blas library. This will require you to reconfigure and rebuild. 
Play around with different number of nodes/cores. Remember to change the mpirun command as well as p and q

### Running

To run using MPI, you'll need to do something that looks like the following:

```bash
mpirun -np <CORES> ./xhpl # CORES = TOTAL cores used, so 3 nodes of 64 is 192
```

Adding OpenMP to this:

- The environment variable `OMP_NUM_THREADS` is the number of OMP threads per rank.

Altogether

```bash
OMP_NUM_THREADS=<THREADS PER RANK> mpirun -np <TOTAL CORES> ./xhpl
```

With multiple physical cores per node, add these additional flags:

```bash
--bind-to socket --map-by socket
```

### Hardware and some calculations

Find the peak FLOPS:

- This is given by `NUMBER OF NODES * NUMBER OF CORE PER NODE * CLOCK SPEED (GHz) * IPC`
- Will give value in GFLOPS.
- Good FLOPS: roughly 65% of peak FLOPS.

To get information about CPU, use `lscpu`:
  - **Base clock speed**, not boost clock speed

To diagnose, use `top` to check load and CPU usage:
  - To sort: hit `f`, select with arrow keys the desired value to sort by, hit `s` and `<esc>`.

Filesystem:
  - Should be NFS (shared)

Scheduler:
  - use `slurm`. See [below](#SLURM-syntax).

MPI libraries:
  - OpenMPI
  - IntelMPI
  - MPICH

Compilers:
  - gcc
  - Clang
  - LLVM
  - Intel (good with IntelMPI)

Networking:
  - Needs to have **low-latency** and **high-bandwidth**
  - Ethernet (1-10 Gb)
  - Infiniband (better than ethernet for performance)

CPU vs. GPU:
  - CPU cheaper and easier to optimize.
  - GPU harder to configure.

### Results

The result you're looking for is the value towards the bottom that says GFLOP/s. That's pretty much the only value we care about

### SLURM syntax

Be aware of `nodes` vs. `ntasks`

```bash
#!/bin/bash
#SBATCH --time=0-0:01:00     # d-hh:mm:ss
#SBATCH --nodes=1            # NUMBER OF NODES
#SBATCH --ntasks=4           # TOTAL CORES
#SBATCH --account=BLAH       # probably given
#SBATCH --chdir=BLAH         # probably share a parent dir with everything in there
#SBATCH --job-name=BLAH      # hpl_bristol something there
#SBATCH --output=BLAH        # both stdout and stderr in here
```
