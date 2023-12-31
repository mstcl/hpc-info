## GROMACS (mdrun)

### Getting good performance on a single node

([source](https://manual.gromacs.org/current/user-guide/mdrun-performance.html#running-mdrun-within-a-single-node))

- thread-MPI (enabled by default in GROMACS): use native, hardware threads on a single node but on a single node, more efficiently.
  - real MPI and thread-MPI from the user perspective looks almost the same
  - GROMACS refers MPI ranks to mean either kind (MPI & thread-MPI)
  - external MPI runs more slowly than thread-MPI
- diagnostics: at runtime, it will put in log/stdout/stderr to inform the user about their choices and consequences

#### Environment variables and MPI

- `OMP_NUM_THREADS`: number of OpenMP threads, we control this
- `mpirun` before `./gmx`: will use external MPI

#### Relevant flags

- `-nt`: number of threads (whether thread-MPI ranks or OpenMP threads within ranks depends on other settings)
- `-ntmpi`: number of thread-MPI ranks to use. Default is one rank per core.
- `-ntomp`: number of OpenMP threads per rank (honors `OMP_NUM_THREADS`), max 64.
- `-npme`: number of ranks to dedicate to long-ranged component of PME. Keep to 2?
- `-ntomp_pme`: number of separate PME ranks, default copies from `-ntomp`.
- `-pin`: attempt to set affinity of threads to cores. Keep to "on".
- `-nb`: if no GPU, set to "cpu".

#### Notes

- MPI might be more performant than OpenMP due to less memory contension.
- PME (Particle-mesh Ewald) ranks, when separate, seems to give better performance.

### Building GROMACS

([source](https://manual.gromacs.org/current/install-guide/index.html#doing-a-build-of-gromacs))

#### First steps

```bash
tar xfz gromacs-2023.3.tgz
cd gromacs-2023.3
mkdir build-gromacs  # or whatever directory, but should be a subdirectory
cd build-gromacs
cmake ..
```

#### Prerequisite & notes

- minimum GNU gcc 9.
- pass multiple options at once.
- use `ccmake` after `cmake ..` returns to see all the settings that were chosen.
- most options have `CMAKE_` and `GMX_` prefixes.

#### Flags

- Help GROMACS find right libraries and binaries external to it.
  - `-DCMAKE_INCLUDE_PATH` for header files
  - `-DCMAKE_LIBRARY_PATH` for libraries
  - `-DCMAKE_PREFIX_PATH` for header, libraries and binaries (e.g. `/usr/local`). For example, `which hwlock` and stick that in.
-`-DCMAKE_INSTALL_PREFIX`: path where GROMAC installs, and places headers, binaries and libraries. This is the root of the GROMACS installation.
- `-DGMX_DOUBLE=off`: turn off double precision as it's slower.
- MPI: `-DGMX_MPI=ON` (this binary is `gmx_mpi`)
- FFT library
  - `-DGMX_BUILD_OWN_FFTW=ON` to let GROMACS build FFTW from source (this is good enough)
  - `-DGMX_FFT_LIBRARY=<your library like fftw3> -DFFTWF_LIBRARY=<path to library>`
- If `hwloc` is installed: `-DGMX_HWLOC=ON` (to improve runtime detection of hw capabilities).
- SIMD: if in doubt, choose the lowest number you think might work and see what mdrun says (highest value leads to performance loss on processors like Skylake and 1st-gen Zen).
  - run `lscpu` and look for "Flags".
  - set with `-DGMX-SIMD=<value>`
- BLAS: `-DGMX_BLAS_USER=<path to your BLAS>`
- LAPACK: `-DGMX_LAPACK_USER=<path to your LAPACK>`

#### Final step for installation

With multi-core run with

```bash
make -j <Number of processors>
make check
make install
```

Source their script `GMXRC` before running it! (Stick this in SLURM)

```bash
source /your/installation/prefix/here/bin/GMXRC
```

#### Overview

1. Get the latest version of your C and C++ compilers.
2. Check that you have CMake version 3.18.4 or later.
3. Get and unpack the latest version of the GROMACS tarball.
4. Make a separate build directory and change to it.
5. Run cmake with the path to the source as an argument
6. Run make, make check, and make install
7. Source GMXRC to get access to GROMACS


## Lustre acronyms

- **MDS** – Manages filenames and directories, file
stripe locations, locking, ACLs, etc.
- **MDT** – Block device used by MDS to store
metadata information
- **OSS** – Handles I/O requests for file data
- **OST** – Block device used by OSS to store file data.
Each OSS usually serves multiple OSTs.
- **MGS** – Management server. Stores configuration
information for one or more Lustre file systems.
- **MGT** - Block device used by MGS for data storage
- **LNET** - (Lustre Networking) provides the underlying communication infrastructure


## General information on HPC

Types of benchmarking

* IOZONE: i/o speed
  * choose big number to read/write
  * read > write
  * care about the time it takes to read/write.vegetarian 
* HPL: see [here](#basic-high-performance-linpack-guide)
* IPM:
* STREAM: memory
  * same as IOZONE but uses memory instead
  * bandwidth vs. size affected by cache (L1, L2, L3, RAM)
  * L1 is the best so we want as much in L1. L1 has split instruction and data cache.
  * L2, L3, and RAM are shared, which can be bottlenecked.


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

- `#N` declares the number of `N` values. Set it to 1.

**Things to consider:**
To check whether its working, set `N` to something low (10,000 or less) and the run should be really fast (but probably not a good result).
Turn this into a slurm script and increase `N` to what the website suggests (and maybe even a little bit higher!)
You can also try changing out compiler/mpi/blas library. This will require you to reconfigure and rebuild. 
Play around with different number of nodes/cores. Remember to change the mpirun command as well as `p` and `q`

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

If in doubt, use `OMP_NUM_THREADS=1`.

* 1 core per rank, and set `p` and `q` so that `p * q` to be equal to the total number of cores.

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

The result you're looking for is the value towards the bottom that says GFLOPs. That's pretty much the only value we care about

## SLURM syntax

Be aware of `nodes` vs. `ntasks`

Documentation: [https://slurm.schedmd.com/sbatch.html](https://slurm.schedmd.com/sbatch.html)

```bash
# submit.sbat or
#!/bin/bash
#SBATCH --time=0-0:01:00     # d-hh:mm:ss
#SBATCH --nodes=1            # NUMBER OF NODES
#SBATCH --ntasks=4           # TOTAL CORES
#SBATCH --account=BLAH       # probably given
#SBATCH --chdir=BLAH         # probably share a parent dir with everything in there
#SBATCH --job-name=BLAH      # hpl_bristol something there
#SBATCH --output=BLAH        # both stdout and stderr in here

# Rest of script here
```

CLI things to note:

`squeue -u <USERNAME>`

Submit scripts:

`sbatch /path/to/script`
