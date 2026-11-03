---
title: ABINIT build system
date: '2026-10-08 16:00:00'
permalink: /post/abinit-build-system.html
layout: post
published: true
---



# ABINIT build system

> **Status, 2026-11-03.** This report describes ABINIT branch `cray_autodetect` at commit `6cd90d43b2`, eight commits on top of upstream develop (section 3). Besides the paths of the external libraries, configure needs no argument on any of the toolchains tested: it finds the compilers, MPI, linear algebra and FFT by itself.
>
> - Workstation, three EasyBuild 2025b toolchains, with the libraries from the fallbacks, from EasyBuild or from both: all seven builds succeed and pass four smoke tests (section 6).
> - Lucia, six toolchains (four Cray PE, two EasyBuild 2025a), with the fallbacks at `b6aa2ef`: all six builds succeed and pass every check of `check.sh` (section 5.3).
>   - Of the seven run-time smoke tests, all pass with GCC and with Intel, both on the Cray PE and with EasyBuild.
>   - CCE gives two small numerical differences beyond the tolerances, which come from its default floating-point optimizations (section 5.6).
>   - AOCC crashes in every calculation with `useylm 1`, PAW included (section 5.4). The cause is a bug of the AOCC 4.1 optimizer, and a second bug of that optimizer changes an output (section 5.6).

Versions:

- ABINIT 10.9.3 (development), branch `cray_autodetect`, commit `6cd90d43b2`: the eight commits of section 3 on top of upstream develop `a0d07ec4c5`.
- External libraries:
  - Lucia: abinit-fallbacks `b6aa2ef` (`my_develop`), one static installation per toolchain (`$FB` below);
  - workstation: abinit-fallbacks `b6aa2ef` (`my_develop`), and EasyBuild modules.

  Packages, versions and build: see [abinit-fallbacks build system](/post/abinit-fallbacks-build-system.html).
- Machines:
  - Lucia (Cenaero), debug nodes with 2 × AMD EPYC 7763 (Zen 3, AVX2, no AVX-512), RHEL 8, Slurm 25.11;
  - a workstation with an Intel Core Ultra 7 265 (20 cores), 30 GB of memory, Ubuntu 24.04.5.

# 1. Goal and rules

For each of the six compiler toolchains available on Lucia, build ABINIT against the libraries built with abinit-fallbacks, with the **smallest possible set of configure arguments**. Section 6 does the same on a workstation with EasyBuild toolchains, where part of the libraries can also come from EasyBuild modules.

- On Lucia, only the **compiler, MPI and math library** come from cluster modules, plus `cray-fftw` on the Cray toolchains. HDF5, netCDF, LibXC, ELPA, Wannier90, XMLF90, libPSML and BigDFT come from the fallbacks, never from `cray-hdf5`, `cray-netcdf` or EasyBuild modules.
- Everything runs in a **clean environment**: `env -i`, then `source /etc/profile` (only for the `module` command), then the module lines of the toolchain.
- Every configure argument beyond the library paths must be justified by a failing attempt without it. Each attempt's `config.log` is kept.
- On Lucia, heavy work runs as Slurm jobs on the debug partition.

# 2. The six toolchains

| Name | Modules | Compilers | MPI | Math library |
|---|---|---|---|---|
| `cray_cray` | `Cray/24.07` `PrgEnv-cray/8.4.0` | CCE 18.0.0 | Cray MPICH 8.1.30 | Cray LibSci 24.07, cray-fftw 3.3.10.8 |
| `cray_gnu` | `Cray/24.07` `PrgEnv-gnu/8.4.0` | GCC 13.3 | Cray MPICH 8.1.30 | Cray LibSci 24.07, cray-fftw 3.3.10.8 |
| `cray_intel` | `Cray/24.07` `PrgEnv-intel/8.4.0` | icx 2022.2 + ifort 2021.7 | Cray MPICH 8.1.30 | Cray LibSci 24.07, cray-fftw 3.3.10.8 |
| `cray_aocc` | `Cray/24.07` `PrgEnv-aocc/8.4.0` | AOCC 4.1 (clang 16 + classic flang) | Cray MPICH 8.1.30 | Cray LibSci 24.07, cray-fftw 3.3.10.8 |
| `eb_intel` | `EasyBuild/2025a` `intel/2025a` | icx/ifx 2025.1 | Intel MPI 2021.15 | MKL 2025.1 |
| `eb_foss` | `EasyBuild/2025a` `foss/2025a` | GCC 14.2 | OpenMPI 5.0.7 | FlexiBLAS 3.4.5 / OpenBLAS 0.3.29, ScaLAPACK 2.2.2 |

Each build directory has an `env.sh` with exactly these lines, for example:

```bash
module --force purge          # also unloads the sticky Cray/ and EasyBuild/ modules
module load Cray/24.07
module load PrgEnv-gnu/8.4.0
module load cray-fftw/3.3.10.8   # Cray toolchains only
```

`cray-fftw/3.3.10.8` is the only version installed. Because `craype-x86-milan` is loaded, the module selects the Milan build of FFTW.

# 3. Changes to ABINIT

| Commit | Change | Section |
|---|---|---|
| `8e703f5ffa` | Fix the compilation with AOCC 4.1 (classic flang) | 3.2 |
| `7087cbaa14` | Detect the Cray PE compilers, Cray LibSci and AOCC flang in configure | 3.1, 3.2 |
| `15456e01e3` | Pass an `xc_info` object to `PSolver` | 3.3 |
| `7f3fa29eb3` | Find LAPACK in OpenBLAS and BLACS in ScaLAPACK | 3.4 |
| `77acf5425e` | `-march=native` with the LLVM compilers, except on AArch64 | 3.4 |
| `a84e3a012d` | Look for OpenBLAS with the LLVM compilers | 3.4 |
| `f7fb136e1c` | Linear algebra Fortran flags in all directories with LLVM Flang | 3.4 |
| `6cd90d43b2` | HDF5 high-level library of CMake builds | 3.4 |

## 3.1 Cray Programming Environment

Commit `7087cbaa14` lets configure find everything a Cray PE provides:

- **Compilers** (`config/m4/arch-mpi.m4`). By default, configure searches `PATH` for MPI wrappers (`config/m4/sd_arch_mpi.m4`), and in every PrgEnv it finds cray-mpich's own `mpicc`/`mpic++`/`mpifort`. Those neither link LibSci nor add the `craype-x86-milan` target options. When `CRAYPE_VERSION` and `PE_ENV` are set and `cc`/`CC`/`ftn` exist, configure now uses the craype wrappers unless `CC`/`CXX`/`FC` are given:

  ```
  checking for a Cray Programming Environment... yes (PrgEnv: GNU)
  configure: using the Cray compiler wrappers: CC=cc CXX=CC FC=ftn
  ```

- **`libsci` linear-algebra flavor** (`sd_math_linalg*.m4`), tried first by `auto` on a Cray PE, or selected with `--with-linalg-flavor=libsci`. Without it, the automatic detection picks the system OpenBLAS (GNU), MKL (Intel, since PrgEnv-intel ships one) or nothing (CCE, AOCC). The flavor is listed in `configure --help` and in `doc/build/config-template.ac9`.
  - With the wrappers, no library has to be named: BLAS, LAPACK, BLACS and ScaLAPACK are all "none required".
  - With other compilers (e.g. `FC=mpifort`), the `libsci_<PrgEnv>` and `libsci_<PrgEnv>_mpi` libraries are linked explicitly, but only when they match the compiler in use.
- **BigDFT link order** (`sd_bigdft.m4`). `libyaml.a` is now passed as `-Wl,<path>/libyaml.a`. The Cray `ftn` driver moves plain file arguments in front of all `-l` options. That put `libyaml.a` *before* `libbigdft-1.a`, so with CCE every BigDFT link failed with `undefined reference to yaml_parser_initialize`.

## 3.2 AOCC 4.1 (classic flang)

AOCC's `ftn --version` prints `AMD clang version 16.0.3 (CLANG: AOCC_4.1.0...)`. That matches ABINIT's LLVM-flang test, and AOCC flang is still treated as LLVM because it needs the LLVM settings. Commit `7087cbaa14` (`lang-fortran.m4`, `configure.ac`) adapts two of them:

- `-ffixed-line-length=132` becomes `-Mextend` in its Fortran hints. Classic flang rejects the former (`clang-16: error: unknown argument: '-ffixed-line-length=132'`), and configure stops with "cannot compile a simple Fortran program".
- The linear-algebra Fortran flags (the ELPA modules) are added to `FCFLAGS` for every directory. Classic flang needs module files that are only used *indirectly* (`elpa_api.mod`), and ABINIT otherwise passes them only to the directories that use linear algebra directly. Commit `f7fb136e1c` extends this to LLVM Flang (section 3.4).

Commit `8e703f5ffa` rewrites three places of ABINIT's own sources that classic flang cannot compile:

| File | Problem | Fix |
|---|---|---|
| `shared/libpaw/src/m_paw_atom_solve.F90` | `type(logical)` (Fortran 2008 `TYPE(intrinsic-type)`) is not supported: `F90-S-0155-Derived type has not been declared - logical` | `logical` |
| `src/62_ctqmc/m_CtqmcInterface.F90` | Bug in flang's integrated preprocessor: the nested macro `MALLOC` → `ABI_MALLOC` comes out as `C(this%Hybrid_chains,(this%num_chains))` | `ABI_MALLOC` directly |
| `src/48_diago/m_chebfi2.F90` (3 places) | Same preprocessor bug inside `ABI_MALLOC_IFNOT` | `if (.not. allocated(x)) then; ABI_MALLOC(x,...); endif` |

The preprocessor bug depends on the data: the same macros expand correctly in hundreds of other places, and in small test files.

## 3.3 `PSolver` argument

Commit `15456e01e3` fixes a real bug that only CCE detected. `m_mklocl_realspace.F90` called BigDFT's `PSolver` with the integer `0` where the interface expects a `type(xc_info)` (`ftn-1279`). The other compilers accepted it, so `PSolver` read an integer as an `xc_info` object. It now passes an object initialised with `xc_init(xc, 0, XC_ABINIT, 1)`.

## 3.4 OpenBLAS + ScaLAPACK, LLVM Flang and CMake HDF5

Five commits, each found with a failing build on the workstation (section 6):

- **LAPACK in OpenBLAS** (`7f3fa29eb3`, `sd_math_linalg_core.m4`). With OpenBLAS, LAPACK was always looked for as `-llapack`. OpenBLAS usually includes LAPACK, and EasyBuild installs no `liblapack`, so the serial detection failed; where a system `liblapack` exists, that reference LAPACK would be linked instead. OpenBLAS's own LAPACK is now used when present, and `-llapack` only when it is missing. This is limited to the `openblas` flavors, because IBM ESSL has routines with LAPACK names but other arguments.
- **BLACS in ScaLAPACK** (same commit). The `netlib` flavor looked for BLACS only as `-lblacs -lblacsCinit -lblacsF77init`, the layout of ScaLAPACK 1.x. ScaLAPACK ≥ 2.0 includes BLACS. When these libraries do not work, BLACS is now looked for in the ScaLAPACK libraries. Before, they also stayed in `LIBS` after the failed check, so the ScaLAPACK check failed as well. With these two changes, an OpenBLAS + ScaLAPACK stack such as EasyBuild's `foss` needs no `--with-linalg-flavor` (sections 5.3 and 6).
- **LLVM optimization flags** (`77acf5425e`, `config/optim/*_llvm_default-optim.conf`). The standard flags were `-mtune=native -mcpu=native`, which clang and flang only accept on AArch64. On x86_64, LLVM 20 stops with `unsupported option '-mcpu='`, and configure fails at its first link test ("linking to Fortran libraries from C fails"). The LLVM compilers now get `-march=native`, as GNU does, and AArch64 keeps the previous flags. AOCC uses these flags too, so `cray_aocc` gets `-O2 -march=native`.
- **Linear algebra with LLVM Flang** (`a84e3a012d`, `sd_math_linalg.m4`). The automatic detection only had sequences for the GNU and Intel Fortran compilers. With LLVM Flang, only `netlib` (`-lblas`) was tried, so OpenBLAS was never found. configure only warned, and the link of the executables failed. OpenBLAS (directly and with pkg-config) and ATLAS are now tried first, as for GNU, but not MKL, which only supports the GNU and Intel compilers.
- **ELPA modules with LLVM Flang** (`f7fb136e1c`, `configure.ac`). Like classic flang, LLVM Flang needs the module files used indirectly: `src/41_geometry/m_supercell.F90` failed with `Cannot parse module file for module 'elpa'`. The linear-algebra Fortran flags now go to every directory with any LLVM Fortran compiler, not only AOCC.
- **HDF5 built with CMake** (`6cd90d43b2`, `sd_io_hdf5.m4`). `h5pcc -showconfig` prints `High-level library: ON` for an HDF5 built with CMake, as EasyBuild's is, while configure only accepted `yes`. Without `-lhdf5_hl`, a static netCDF built against that HDF5 (by the fallbacks) failed to link (`undefined reference to H5DSattach_scale`, …), and configure disabled netCDF. `ON` is now accepted, as for the shared-library check just below. The Fortran high-level library keeps its previous handling, since its name differs between CMake (`libhdf5_hl_fortran`) and autotools (`libhdf5hl_fortran`) builds.

# 4. Configure arguments

The same command for every toolchain of Lucia. It contains only the libraries to use, and no `--prefix` (ABINIT is built in place):

```bash
../configure \
  --with-hdf5=$FB/hdf5/default \
  --with-netcdf=$FB/netcdf4/default \
  --with-netcdf-fortran=$FB/netcdf4_fortran/default \
  --with-libxc=$FB/libxc/default \
  --with-elpa=$FB/elpa/default \
  --with-wannier90=$FB/wannier90/default \
  --with-xmlf90=$FB/xmlf90/default \
  --with-libpsml=$FB/libpsml/default \
  --with-bigdft=$FB/bigdft/default
```

No toolchain needs another argument:

- the four Cray toolchains get LibSci through the craype wrappers (section 3.1), and FFTW3 from the `cray-fftw` module (section 5.5);
- `eb_intel` gets MKL, with its DFTI FFT;
- `eb_foss` gets OpenBLAS with ScaLAPACK (section 3.4), and EasyBuild's FFTW3 through pkg-config.

# 5. Results on Lucia

On 2026-11-03, the six toolchains were built and tested with commit `6cd90d43b2` and the fallbacks at `b6aa2ef`: full builds (T1), a few run-time smoke tests (T2), and the FFTW library loaded at run time on Cray (T3). This is not a run of the test suite. Section 5.6 then traces the failures to their causes.

## 5.1 How the tests were run

The rules of section 1 apply: clean environment, no argument besides the library paths, heavy work in Slurm jobs on the debug partition.

- **Source tree** (`$ABI` below): branch `cray_autodetect` at `6cd90d43b2`, with a clean `git status`.
  - The build system was regenerated once, before the builds, with `config/scripts/makemake`. Then `aclocal.m4`, `config.h.in`, `configure` and the `Makefile.in` files were touched in that order (section 7).
  - No `git pull` was done during the builds, and no `make` reran the Autotools.
- **Fallbacks.** `FB=~/program/abinit-fallbacks/_build_<toolchain>`: the six installations of commit `b6aa2ef`. For each of them, the fallbacks' `check.sh` reports `RESULT: OK`.
- **Build directories.** One new, empty directory per toolchain, `$ABI/_build_<toolchain>`. Each holds `env.sh` (section 2), `build.sh`, `check.sh` (section 5.2) and `tests.sh`.
- **Python.** The default `python3` of RHEL 8 is 3.6, while `runtests.py` needs Python ≥ 3.10 and `makemake` needs ≥ 3.11.
  - `runtests.py` is called by the absolute path of a Python 3.12 from a Conda environment in `$HOME`. No Python module is loaded into the toolchain environment.
  - For `makemake`, a directory with only two links to that interpreter, `python3` and `python`, goes in front of `PATH`. Both are needed because `abisrc.py`, which `makemake` runs, starts with `#!/usr/bin/env python`.
- **Slurm variables.** `env -i` also removes the `SLURM_*` variables, and without them `srun` inside a job asks for a new allocation. The test jobs therefore save these variables before cleaning the environment, and source them again after `env.sh`.

**T1: full builds.** One Slurm job per toolchain, on a whole debug node. In each build directory, `build.sh`:

```bash
source /etc/profile
source ./env.sh
FB=~/program/abinit-fallbacks/_build_<toolchain>
../configure \
  --with-hdf5=$FB/hdf5/default --with-netcdf=$FB/netcdf4/default \
  --with-netcdf-fortran=$FB/netcdf4_fortran/default --with-libxc=$FB/libxc/default \
  --with-elpa=$FB/elpa/default --with-wannier90=$FB/wannier90/default \
  --with-xmlf90=$FB/xmlf90/default --with-libpsml=$FB/libpsml/default \
  --with-bigdft=$FB/bigdft/default > configure.out 2>&1 || exit 1
make -j 64 > make.out 2>&1
echo "make exit status: $?" >> make.out
./check.sh > check.out 2>&1
```

and the job script:

```bash
#!/bin/bash
#SBATCH --partition=debug
#SBATCH --nodes=1
#SBATCH --exclusive
cd $ABI/_build_<toolchain>
env -i HOME=$HOME USER=$USER LOGNAME=$LOGNAME TERM=$TERM PATH=/usr/bin:/bin \
    bash --noprofile --norc ./build.sh
```

**T2: smoke tests.** After T1, one Slurm job per toolchain with 4 tasks:

| Test | MPI processes | Exercises |
|---|---|---|
| `v1:01` | 1 | a plane-wave ground state, with the FFT of the build (FFTW3, or DFTI on `eb_intel`) |
| `libxc:01` | 1 | LibXC |
| `bigdft:00` | 1 | BigDFT's Poisson solver (`icoulomb 1`, GTH pseudopotential), through `createIonicPotential_new`, whose `PSolver` call is fixed by `15456e01e3` |
| `wannier90:00` | 1 | Wannier90 (`prtwant 2`) |
| `psml:05` | 1 | libPSML and XMLF90 (reads `14_Si_oncv.psml`) |
| `paral:24` | 4 | `paral_kgb 1` with `use_slk 1`: BLACS, ScaLAPACK and ELPA, which the distributed eigensolver of `m_slk` uses when available. It would stop in `FAKE BLACS_GRIDINIT` if the ScaLAPACK stubs of BigDFT were linked (fallbacks report, section 4) |
| `mpiio:22` | 4 | MPI-IO with `paral_kgb 1` |

In each build directory, `tests.sh`:

```bash
source /etc/profile
source ./env.sh
source ./slurm_env.sh                # the SLURM_* variables of the job
PY=<absolute path of a Python >= 3.10>
MPI_ARGS=--mpi=cray_shasta           # eb_intel: --mpi=pmi2, eb_foss: --mpi=pmix (section 7)
export LD_LIBRARY_PATH=$CRAY_LD_LIBRARY_PATH:$LD_LIBRARY_PATH   # Cray only: Milan FFTW (section 5.5)
export I_MPI_PMI_LIBRARY=/usr/lib64/libpmi2.so                   # eb_intel only
$PY ../tests/runtests.py -b . -w tests_seq -q --use-srun --mpi-args="$MPI_ARGS" \
    v1:01 libxc:01 bigdft:00 wannier90:00 psml:05
$PY ../tests/runtests.py -b . -w tests_mpi -q --use-srun --mpi-args="$MPI_ARGS" -n 4 \
    paral:24 mpiio:22
```

and the job script:

```bash
#!/bin/bash
#SBATCH --partition=debug
#SBATCH --nodes=1
#SBATCH --ntasks=4
cd $ABI/_build_<toolchain>
export -p | grep '^declare -x SLURM_' > slurm_env.sh
env -i HOME=$HOME USER=$USER LOGNAME=$LOGNAME TERM=$TERM PATH=/usr/bin:/bin \
    bash --noprofile --norc ./tests.sh
```

With `--use-srun`, the serial tests also run with `srun -n 1`. A test counts as passed when `tests_seq/results.json` or `tests_mpi/results.json` gives `succeeded` or `passed`. A `skipped` test would mean that a CPP option is missing, so it counts as a failure.

**T3: FFTW at run time on Cray.** This runs in the same job as T2, after `tests.sh`, on the four Cray builds:

- `ldd src/98_main/abinit | grep fftw`, with and without the `LD_LIBRARY_PATH` line of `tests.sh`;
- `v1:01` and `paral:24` again, without that line, in other work directories (`tests_rome_seq`, `tests_rome_mpi`).

A fifth build, `_build_cray_gnu_rpath`, is `cray_gnu` with `export CRAY_ADD_RPATH=yes` before configure. Its test job runs `v1:01` without `LD_LIBRARY_PATH`.

## 5.2 What `check.sh` verifies

Every build ends with a `check.sh` that verifies:

1. No `.ac9` configuration file was read: `not loading options (no config file available)` in `config.log`.
2. The compilers are the expected ones: `cc`/`CC`/`ftn` on Cray, the Intel or OpenMPI wrappers on EasyBuild, and the first line of `--version` gives the expected compiler.
3. MPI works for C, C++ and Fortran, and MPI-IO is enabled. The result of the MPI-IO test is read from `abi_mpi_io_enable` and from the configure summary (`MPI-IO enabled : yes`). The line `checking whether to build MPI I/O code...` cannot be used: it prints the value of the option (`auto`), not the result.
4. The math library is the cluster's:
   - linear algebra, ScaLAPACK and ELPA are all `yes`;
   - `ldd abinit` shows `libsci_<x>(_mpi)` on Cray, MKL on `eb_intel`, and EasyBuild OpenBLAS/FlexiBLAS + ScaLAPACK on `eb_foss`;
   - never `/usr/lib64/libopenblas*`.
5. Every `-I`/`-L` of HDF5, netCDF, LibXC, ELPA, Wannier90, XMLF90, libPSML and BigDFT points into `$FB`. None of them, nor `libyaml`, appears as a shared library in `ldd abinit`.
6. `config.h` defines `HAVE_HDF5 HAVE_NETCDF HAVE_NETCDF_FORTRAN`, and both `config.h` and `abinit -b` list `HAVE_MPI HAVE_MPI_IO HAVE_HDF5_MPI HAVE_NETCDF_MPI HAVE_NETCDF_FORTRAN_MPI HAVE_LIBXC HAVE_LINALG_SCALAPACK HAVE_LINALG_ELPA HAVE_WANNIER90 HAVE_LIBPSML HAVE_XMLF90 HAVE_BIGDFT`.
7. `make` exits with 0.
8. On Cray only, FFTW comes from the `cray-fftw` module:
   - the FFT flavor is `fftw3`;
   - the FFTW3 flags point to `$FFTW_ROOT`;
   - `abinit -b` lists `HAVE_FFTW3`;
   - `ldd abinit` resolves `libfftw3*` to a `cray-fftw` file, both without `LD_LIBRARY_PATH` and with `LD_LIBRARY_PATH=$CRAY_LD_LIBRARY_PATH`.

It also checks the points specific to each toolchain (section 5.3), and prints the linear algebra and FFT flavors and the `ldd` lines of both libraries.

## 5.3 T1: full builds

All six builds succeed, with no configure argument besides the library paths of section 4. `check.sh` reports `RESULT: OK` (checks 1–8) on every one:

| Toolchain | Compilers found by configure | FC optimization | Build time | Linear algebra | FFT | `ldd abinit`: linear algebra, FFT |
|---|---|---|---|---|---|---|
| `cray_cray` | `cc`/`CC`/`ftn`, CCE 18.0.0 | `-O2 -h nofortran_ptr_alias -h nofortran_ptr_overlap -Oipa0` | 32:47 | `elpa+libsci` | FFTW3 (cray-fftw), + `HAVE_FFTW3_MPI` | `libsci_cray(_mpi).so.6`, `libfftw3*.so.mpi31.3` |
| `cray_gnu` | `cc`/`CC`/`ftn`, GCC 13.3.1 | `-O2 -march=native` | 8:45 | `elpa+libsci` | FFTW3 (cray-fftw), + `HAVE_FFTW3_MPI` | `libsci_gnu(_mpi).so.6`, `libfftw3*.so.mpi31.3` |
| `cray_intel` | `cc`/`CC`/`ftn`, ifort 2021.7 + icx 2022.2 | `-O3` | 21:14 | `elpa+libsci` | FFTW3 (cray-fftw), + `HAVE_FFTW3_MPI` | `libsci_intel(_mpi).so.6`, `libfftw3*.so.mpi31.3` |
| `cray_aocc` | `cc`/`CC`/`ftn`, AOCC 4.1 | `-O2 -march=native` | 27:18 | `elpa+libsci` | FFTW3 (cray-fftw), + `HAVE_FFTW3_MPI` | `libsci_aocc(_mpi).so.6`, `libfftw3*.so.mpi31.3` |
| `eb_intel` | `mpiicx`/`mpiicpx`/`mpiifx`, 2025.1.1 | `-O3` | 10:20 | `elpa+mkl` | DFTI | MKL 2025.1: `libmkl_intel_lp64`, `_sequential`, `_core`, `_scalapack_lp64`, `_blacs_intelmpi_lp64`, `_cdft_core` |
| `eb_foss` | `mpicc`/`mpic++`/`mpifort`, GCC 14.2.0 | `-O2 -march=native` | 10:06 | `elpa+netlib+openblas` | FFTW3 (EasyBuild) | `libopenblas.so.0` (OpenBLAS 0.3.29), `libscalapack.so.2.2` (ScaLAPACK 2.2.2), `libflexiblas.so.3`, `libfftw3(f).so.3` (FFTW 3.3.10) |

The build time is the wall time of the job (configure, `make -j 64` and `check.sh`) on a debug node. On Cray, LibSci and `libfftw3*` come from `/opt/cray/pe/lib64`; section 5.5 shows which FFTW build that gives at run time. In every build, `ldd abinit` lists no library of the fallbacks, which are static, and no `/usr/lib64/libopenblas*`.

- **`cray_aocc`**: the first full build with classic flang and the commits of section 3.4. Every directory compiles.
  - The optimization flags are `-O2 -march=native` (`77acf5425e`).
  - `config.log` shows `adding the linear algebra Fortran flags to FCFLAGS for Flang` (`f7fb136e1c`).
  - LibSci is still tried first: the serial sequence is `libsci openblas atlas netlib openblas_pkg`, with OpenBLAS added for LLVM by `a84e3a012d`.
- **`cray_cray`**: `make.out` has no `ftn-1279` error (section 3.3), and `abinit` links with BigDFT (`libyaml` order, section 3.1).
- **`eb_foss`**: `elpa+netlib+openblas` without `--with-linalg-flavor`. BLACS is first looked for as `-lblacs …`, which ScaLAPACK 2.2.2 does not provide, then inside ScaLAPACK (`7f3fa29eb3`). The libraries are `-lelpa -lscalapack -lopenblas -lpthread`, and `libflexiblas` is only loaded by EasyBuild's ScaLAPACK:

  ```
  checking for the serial linear algebra detection sequence... openblas atlas netlib openblas_pkg
  checking for the MPI linear algebra detection sequence... netlib
  checking for BLACS support in the specified libraries... no
  checking for BLACS support in the specified libraries... yes
  checking for the actual linear algebra flavor... elpa+netlib+openblas
  ```

- **`eb_intel`**: configure picks the Intel MPI wrappers of the LLVM-based compilers (`mpiicx`/`mpiicpx`/`mpiifx`). ScaLAPACK and BLACS come from MKL.

## 5.4 T2: smoke tests

| Test | `cray_cray` | `cray_gnu` | `cray_intel` | `cray_aocc` | `eb_intel` | `eb_foss` |
|---|---|---|---|---|---|---|
| `v1:01` | OK | OK | OK | OK | OK | OK |
| `libxc:01` | OK | OK | OK | OK | OK | OK |
| `bigdft:00` | OK | OK | OK | OK | OK | OK |
| `wannier90:00` | **diff** | OK | OK | **crash** | OK | OK |
| `psml:05` | OK | OK | OK | OK | OK | OK |
| `paral:24` (4 MPI) | **diff** | OK | OK | **crash** | OK | OK |
| `mpiio:22` (4 MPI) | OK | OK | OK | **crash** | OK | OK |

OK means `succeeded`, or `passed` within the tolerances (as `bigdft:00` everywhere and `wannier90:00` where it does not fail). No test was skipped, so every CPP option the tests need is present. All seven tests pass with GCC (Cray PE and `foss`) and Intel (Cray PE and EasyBuild).

**AOCC: crash with `useylm 1`.** `wannier90:00`, `paral:24` and `mpiio:22` stop with a segmentation fault (exit status 139) in the first SCF iteration, at the same place:

```
Caught signal 11 (Segmentation fault: address not mapped to object at address (nil))
 1 m_opernla_ylm_opernla_ylm_()  src/66_nonlocal/m_opernla_ylm.F90:0
 2 m_nonlop_ylm_nonlop_ylm_()    src/66_nonlocal/m_nonlop_ylm.F90:877
 3 m_nonlop_nonlop_()            src/66_nonlocal/m_nonlop.F90:930
 4 m_getghc_getghc_()            src/66_wfs/m_getghc.F90:1096
 5 m_cgwf_cgwf_()                src/67_common/m_cgwf.F90:550
```

- The three tests that crash all use `useylm 1`, so the nonlocal operator goes through `nonlop_ylm` and `opernla_ylm`:
  - `paral:24` and `mpiio:22` are PAW, which implies `useylm 1`;
  - `wannier90:00` uses norm-conserving pseudopotentials, but sets `useylm 1` explicitly.
- The four tests that pass use norm-conserving pseudopotentials with the default `useylm 0`, which goes through `nonlop_pl` instead.
- So the AOCC build cannot run any PAW calculation, nor any calculation with `useylm 1`.
- The compilation of `m_opernla_ylm.F90` prints no warning, and the crash is the same with the other FFTW build (section 5.5).
- The cause is a bug of the AOCC optimizer (section 5.6).

The `.fldiff` files, in `$ABI/_build_cray_aocc/`, only report the missing output: `tests_seq/wannier90_t00/t00.abo.fldiff`, `tests_mpi/paral_t24_MPI4/t24_MPI4.abo.fldiff`, `tests_mpi/mpiio_t22_MPI4/t22_MPI4.abo.fldiff`.

**CCE: numerical differences.** Two tests run to the end, with differences slightly beyond their tolerances:

- `paral:24` is a cell relaxation (`ionmov 3`, `optcell 1`, `ntime 3`). The final pressure is 1.4718 GPa against 1.4729 GPa in the reference. That difference, 1.1e-3, is above `tolabs` = 3e-4. The other values differ less:
  - the total energy, by about 4e-9 Ha;
  - the primitive vectors, 3.3448631 against 3.3448607 bohr;
  - `sigma(1 1)`, −5.0027e-5 against −5.0061e-5 Ha/bohr³.

  The fldiff report is `tests_mpi/paral_t24_MPI4/t24_MPI4.abo.fldiff`.
- `wannier90:00`: 5 lines of the matrix printed for Wannier90 differ. The largest relative difference is −0.007642 against −0.007654, 7.8e-4, above `tolrel` = 4e-4; the absolute difference, 1.4e-5, is within `tolabs`. The fldiff report is `tests_seq/wannier90_t00/t00.abo.fldiff`.

The paths are in `$ABI/_build_cray_cray/`. With GCC and Intel, the same tests stay within these tolerances. The reference files are unchanged. Section 5.6 shows that these differences come from CCE's default floating-point optimizations.

## 5.5 FFTW on Cray: `cray-fftw`

Without the module, configure's FFTW3 test fails (`cannot find -lfftw3_mpi`) and ABINIT falls back to its internal Goedecker FFT. Loading `cray-fftw` is enough, with no change to ABINIT and no configure argument:

- `cray-fftw` adds its `fftw3.pc` to `PKG_CONFIG_PATH`, and ABINIT's FFTW3 macro (`sd_fftw3.m4`) uses pkg-config when it is available:

  ```
  checking for fftw3 via pkg-config... yes
  checking whether the FFTW3 library works... yes
  checking whether the FFTW3 library supports threads... yes
  checking whether the FFTW3 MPI library works... yes
  checking for the actual FFT flavor to use... fftw3
  ```

- The craype wrappers add `-I$FFTW_INC`. They also link the module's libraries, `fftw3_mpi` and `fftw3_threads` included. So the `fftw3-mpi.f03` test passes and `HAVE_FFTW3_MPI` is defined, although the pkg-config flags give only `-lfftw3 -lfftw3f`.
- This works the same with CCE, GCC, Intel and AOCC. ABINIT includes the FFTW Fortran 2003 interface files (`fftw3.f03`, `fftw3-mpi.f03`), which use only `iso_c_binding`. They are therefore independent of the Fortran compiler, and no compiler-specific FFTW build is needed.

One detail matters at run time. The binary is linked against the Milan build (`x86_milan`) because the module is loaded. Without `LD_LIBRARY_PATH`, however, the loader takes `libfftw3*.so.mpi31.3` from `/opt/cray/pe/lib64`, which is listed in `ld.so.conf`. On Lucia those files point to the **`x86_rome`** build of the same version. To run with the Milan build, use Cray's usual setting:

```bash
export LD_LIBRARY_PATH=$CRAY_LD_LIBRARY_PATH:$LD_LIBRARY_PATH
```

T3 confirms this on the four Cray builds, and both variants give the results of T2:

| Build | `libfftw3*` without `LD_LIBRARY_PATH` | with `LD_LIBRARY_PATH=$CRAY_LD_LIBRARY_PATH` | `v1:01`, `x86_rome` | `paral:24`, `x86_rome` |
|---|---|---|---|---|
| `cray_cray` | `x86_rome` | `x86_milan` | OK | **diff** (as in T2) |
| `cray_gnu` | `x86_rome` | `x86_milan` | OK | OK |
| `cray_intel` | `x86_rome` | `x86_milan` | OK | OK |
| `cray_aocc` | `x86_rome` | `x86_milan` | OK | **crash** (as in T2) |

The `cray_gnu` build linked with `CRAY_ADD_RPATH=yes` embeds the module's library path in the binary. Without `LD_LIBRARY_PATH`, `ldd abinit` then resolves every `libfftw3*` to `x86_milan`, `v1:01` passes, and `check.sh` reports `RESULT: OK`.

## 5.6 Diagnosis: compilers, not the source

All the failures of section 5.4 come from the compilers: two optimizer bugs of AOCC 4.1, and the default floating-point optimizations of CCE. None of them points to an error in the ABINIT sources.

**AOCC 1: absent optional argument read in `opernla_ylm`.** This bug causes the three crashes.

- The faulting instruction is `movslq (%rax),%rax`, a 4-byte load through an address taken from the stack slot `0xb30(%rsp)`.
  - The prologue of `opernla_ylm` saves 6 registers and reserves `0xa48` bytes, so argument *k* ≥ 7 is at `rsp+0xa80+8(k−7)`.
  - `0xb30` is therefore argument 29, the optional `qdir`. It is absent in these calls, so its address is null.
- In the source, `qdir` is only used inside `if ((signs==2).and.(choice_==33))`, as `igamma=qdir` then `idelgam=gamma(idelta,igamma)` (`m_opernla_ylm.F90:647` and `:2027`).
  - `nonlop_ylm` requires `qdir` only in that case (`m_nonlop_ylm.F90:456-459`).
  - Passing an absent optional argument to an optional dummy argument is valid Fortran.
- In the code generated at `-O2`, the test `choice_==33` (`cmp $0x21`) only sets a flag (`setne`). The load of `qdir` and the product `3*qdir` for the index of `gamma` (`lea (%rax,%rax,2)`) come next, with no branch in between and before the loops: the optimizer has moved the load out of its condition.
- Experiment: `m_opernla_ylm.F90` was compiled again, alone, at `-O0` or `-O1`, with the compile command of `make.out`. It was linked with all the other objects of the T1 build and run with `runtests.py` as in T2.
  - `paral:24` and `mpiio:22` then give `succeeded`;
  - `wannier90:00` runs to the end.

**AOCC 2: wrong trip count in `write_Mmn`.** With that change, `wannier90:00` still fails, for another reason. Its values agree with GCC and Intel within 2e-6, but the first three rows of the overlap matrix printed by `write_Mmn` (`src/71_wannier/m_wannier_io.F90`) have four elements instead of three. The loop is:

```fortran
do jj=1,3
  if(jj>num_bands(isppol))cycle
  write(msg, '(a,2f11.6,a)') trim(msg),M_matrix(ii,jj,1,1,isppol),' , '
end do
```

`num_bands` is 4 in this test, and at `-O2` AOCC runs `jj` from 1 to 4. A standalone program reproduces it:

```fortran
module m_w
  implicit none
contains
  subroutine top(nb)
    integer, intent(in) :: nb(:)
    integer :: ii, jj
    do ii = 1, 3
      if (ii > nb(1)) cycle
      do jj = 1, 3
        if (jj > nb(1)) cycle
        write(*,'(2(a,i0))') ' ii=', ii, ' jj=', jj
      end do
    end do
  end subroutine top
end module m_w
program p
  use m_w
  implicit none
  call top([4])
end program p
```

- With AOCC 4.1 (`PrgEnv-aocc`, `ftn -O2 -march=native`), it prints `jj=1` to `jj=4` for each `ii`.
- With AOCC at `-O1`, and with GCC at `-O1` and `-O2`, it prints `jj=1` to `jj=3`.
- In `write_Mmn`, the internal file `msg` also appears in the output list of its own `write`, which the Fortran standard does not allow. That is not the cause: the program above has no internal write.
- Any loop of this shape (a `cycle` on a bound larger than the trip count) may be miscompiled at `-O2`, so AOCC may give wrong results elsewhere, where no test looks.

**CCE: floating-point optimizations.** A separate CCE build used the same configure command and, for the Fortran optimization, the flags of T1 plus `-h fp0` (strict IEEE conformance). This was set through `FCFLAGS_OPTIM` with `--with-optim-flavor=custom` (section 7), for this diagnosis only.

- All seven tests of T2 pass. `paral:24` gives the values of the reference to the last printed digit: final pressure 1.4729 GPa, `acell` 6.6897213787, and the same total energies. The line of `wannier90:00` is the one of GCC.
- The first value that differs between the two CCE builds is the PAW compensation charge at the end of the first SCF cycle: 0.223643631497841 with `-h fp0`, 0.223643631504618 with the default flags, a difference of 7e-12.
  - The cell relaxation of `paral:24` amplifies that difference over its three steps.
  - `wannier90:00` stops after two SCF iterations (`nstep 2`). Its wavefunctions are not converged, so its overlap matrix is sensitive to such differences.
- CCE's default level is `-h fp2` (`crayftn(1)`). Above `-h fp1`, it also allows square roots and divisions to be rewritten with hardware reciprocal approximations (`approx`). `-h fp1`, which keeps them exact, was not tested.

These differences are therefore allowed by CCE's default options. They are a bug neither of CCE nor of ABINIT.

# 6. Workstation tests: EasyBuild 2025b

Three EasyBuild 2025b toolchains:

- `foss`: GCC 14.3, OpenMPI 5.0.8, FlexiBLAS 3.4.5 / OpenBLAS 0.3.30, ScaLAPACK 2.2.2, FFTW 3.3.10;
- `intel`: icx/ifx 2025.2, Intel MPI 2021.16, MKL 2025.2;
- `lfoss`: LLVM 20.1.8 (clang + flang), otherwise as `foss`.

Each build runs in the clean environment of section 1. The fallbacks (`b6aa2ef`) are built with no argument, and build only the libraries not provided by the modules loaded. Three scenarios differ in the EasyBuild modules loaded besides the toolchain:

- `fb`: none, so the fallbacks build every library;
- `mix`: HDF5 1.14.6, netCDF 4.9.3, netCDF-Fortran 4.6.2 (all with parallel I/O), LibXC 7.0.0 and ELPA 2025.06.002. The fallbacks build Wannier90, XMLF90, libPSML and BigDFT;
- `partial` (`foss` only): HDF5 and LibXC.

ABINIT then gets `--with-<package>=$FB/<package>/default` for each library installed by the fallbacks, and no other argument. Smoke tests: `v1:01` and `libxc:01` (serial), `paral:24` (`paral_kgb 1` with `use_slk 1`) and `mpiio:22` (MPI-IO) with 4 MPI processes.

All seven builds succeed and pass the four smoke tests (2026-11-03):

| Build | `--with-…` paths | From EasyBuild (pkg-config) | Linear algebra | FFT |
|---|---|---|---|---|
| `foss` fb | 9 | — | `elpa+netlib+openblas` | FFTW3 |
| `foss` mix | 4 | HDF5, netCDF, netCDF-Fortran, LibXC, ELPA | `elpa+netlib+openblas` | FFTW3 |
| `foss` partial | 7 | HDF5, LibXC | `elpa+netlib+openblas` | FFTW3 |
| `intel` fb | 9 | — | `elpa+mkl` | DFTI |
| `intel` mix | 4 | HDF5, netCDF, netCDF-Fortran, LibXC, ELPA | `elpa+mkl` | DFTI |
| `lfoss` fb | 8 (no BigDFT) | — | `elpa+netlib+openblas` | FFTW3 |
| `lfoss` mix | 3 (no BigDFT) | HDF5, netCDF, netCDF-Fortran, LibXC, ELPA | `elpa+netlib+openblas` | FFTW3 |

In every build, `abinit -b` lists `HAVE_MPI_IO HAVE_HDF5_MPI HAVE_NETCDF_MPI HAVE_NETCDF_FORTRAN_MPI HAVE_LIBXC HAVE_LINALG_SCALAPACK HAVE_LINALG_ELPA HAVE_WANNIER90 HAVE_LIBPSML HAVE_XMLF90`, plus `HAVE_BIGDFT` except with LLVM Flang, for which the fallbacks do not build BigDFT. `ldd abinit` lists only libraries of EasyBuild and of the system: no linear algebra from the system, and no library of the fallbacks, which are static.

- **BLAS through OpenBLAS or FlexiBLAS.** With GNU and LLVM, BLACS and ScaLAPACK come from `-lscalapack` (section 3.4). In `fb` and `partial`, ABINIT links `-lelpa -lscalapack -lopenblas`, so its BLAS calls go to OpenBLAS; FlexiBLAS is only loaded by EasyBuild's ScaLAPACK. In `mix`, the pkg-config flags of EasyBuild's ELPA (`-lelpa -lscalapack -lflexiblas`) come before `-lopenblas`, so the BLAS calls go through FlexiBLAS, whose default backend is the same OpenBLAS.
- **One HDF5 for the whole I/O stack.** In `foss` partial, the fallbacks build netCDF against EasyBuild's HDF5, and ABINIT links that static netCDF with EasyBuild's shared HDF5 (`6cd90d43b2`).

# 7. Pitfalls worth knowing

- **Automake rebuild rules.** ABINIT's Makefiles have no "maintainer mode". If a file of `config/m4` is newer than `configure`, every `make` reruns aclocal/autoconf/automake *in the shared source tree*, and concurrent builds collide (`autom4te: cannot rename …`).
  - `aclocal` does not rewrite `aclocal.m4` when only the content of an included m4 file changes. So after `./autogen.sh` or `makemake`, touch `aclocal.m4`, `config.h.in`, `configure` and the `Makefile.in` files, in that order.
- **No `git pull` while builds run.** All the build directories share one source tree:
  - a running `make` that sees a newer `configure.ac` reruns `aclocal` and `autoconf` in the source tree, then `config.status --recheck`, i.e. configure again in the middle of the build. The binary then mixes objects compiled with two different `config.h`;
  - a build that is in configure when `configure` is rewritten stops with `syntax error near unexpected token`.

  After a pull, regenerate the build system with `config/scripts/makemake`, because changes to `config/optim` only reach `configure` that way, not through the rebuild rules. Then rebuild from scratch.
- **MPI launch inside a Slurm job** (Slurm 25.11, no default MPI plugin). Each library needs its own plugin, or the ranks start as separate singletons:
  - Cray MPICH: `srun --mpi=cray_shasta`;
  - Intel MPI: `srun --mpi=pmi2` with `I_MPI_PMI_LIBRARY=/usr/lib64/libpmi2.so`;
  - OpenMPI 5: `srun --mpi=pmix`.
- **Python in a clean environment.** The only Python of the clean environment is the `python3` 3.6 of RHEL 8, and there is no `python` at all.
  - `makemake` needs Python ≥ 3.11.
  - `makemake` runs `abisrc.py`, which starts with `#!/usr/bin/env python`. Without a `python` in `PATH`, `makemake` stops with `The abisrc.py script returned a non-zero exit status`, and `abisrc.stderr` contains `/usr/bin/env: 'python': No such file or directory`.
  - `runtests.py` runs `tests/pymods/diff.py`, also through `env python`, when a comparison fails. Without `python`, the log shows the same message. Only the HTML diff is missing; the result of the test is not affected.

- **No `make -n` in a build tree.** GNU make remakes its makefiles even with `-n`.
  - `make -n -B abinit` therefore really runs `make am--refresh`, because `-B` makes every file out of date, the Makefiles included. That runs aclocal, automake and autoconf in the shared source tree, then `config.status --recheck`, i.e. configure again.
  - To get a compile or link command, read it in `make.out`: the builds of section 5.1 print full command lines.
- **Optimization flags from the environment.** configure uses `FCFLAGS_OPTIM`, `CFLAGS_OPTIM` and `CXXFLAGS_OPTIM` only with `--with-optim-flavor=custom`.
  - Without that option, it ignores them and does not say so.
  - `--with-optim-flavor=yes` is rejected; the allowed values are `none`, `custom`, `safe`, `standard` and `aggressive`.
  - The CCE build of section 5.6 sets all three variables: the Fortran flags of T1 plus `-h fp0`, and `-O2` for C and C++ as in T1.

# 8. Open points

- **AOCC optimizer bugs** (section 5.6). Until they are fixed, `cray_aocc` builds need a workaround:
  - compile the files concerned at `-O1`, as configure already does for `src/70_gw` with CCE (`fcflags_opt_70_gw`);
  - or avoid the patterns in the source, for example by passing a defined `qdir` to `opernla_ylm`.

  The loop bug may hit other loops: a wider run of the test suite with AOCC would show how many.
- **CCE: tolerances or floating-point level.** `paral:24` and `wannier90:00` pass with `-h fp0` (section 5.6). There are two options:
  - widen the tolerances of these two tests;
  - or build with a stricter floating-point level for CCE. The cost of `-h fp0` or `-h fp1` in run time has not been measured.
- **No linear algebra is only a warning.** When nothing works, configure prints `linear algebra does not work` and goes on (policy `warn`), so the error only appears when the executables are linked, after the whole build.
- **`FCFLAGS_HINTS`** cannot override the vendor hints. `configure.ac` applies the user's value (`ABI_ENV_RECALL`, line 136) *before* computing the vendor hints (`ABI_FC_HINTS`, line 403), which overwrite it. With the AOCC changes of section 3.2, Lucia does not need it.
- **Upstream reports to AMD**: the preprocessor bug of section 3.2 and the two optimizer bugs of section 5.6. The loop bug can be reported with the program of section 5.6; the `opernla_ylm` bug has no small reproducer yet.
- **FFTW variant at run time on Cray.** By default, the loader picks the `x86_rome` build of `cray-fftw` (section 5.5). Linking with `CRAY_ADD_RPATH=yes` gives the Milan build with no `LD_LIBRARY_PATH`; this was tested with `cray_gnu`. What remains to decide is whether to only document it or to use it by default for the Cray builds.
- **FFTW3 threads.** OpenMP is off in these builds, so ABINIT uses `fftw3`, not `fftw3-threads`, although `cray-fftw` provides the threaded libraries.
