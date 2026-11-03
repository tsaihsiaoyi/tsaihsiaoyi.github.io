---
title: abinit-fallbacks build system
date: '2026-10-08 15:00:00'
permalink: /post/abinit-fallbacks-build-system.html
layout: post
published: true
---



# abinit-fallbacks build system

> **Status, 2026-11-03.** This report describes abinit-fallbacks at commit `b6aa2ef` of `my_develop`. With no configure argument, it detects the compilers, MPI and linear algebra of the loaded modules, uses the external libraries already available, and builds the missing ones from source as static libraries. ABINIT itself, built against these libraries, is described in [ABINIT build system](/post/abinit-build-system.html).

Versions:

- abinit-fallbacks: branch `my_develop` of the fork [tsaihsiaoyi/abinit-fallbacks](https://github.com/tsaihsiaoyi/abinit-fallbacks), commit `b6aa2ef`.
- Machines:
  - Lucia (Cenaero), debug nodes with 2 × AMD EPYC 7763 (Zen 3, AVX2, no AVX-512), RHEL 8, Slurm 25.11;
  - a workstation with an Intel Core Ultra 7 265 (20 cores), 30 GB of memory, Ubuntu 24.04.5.

## 1. Goal and rules

For different compiler toolchains available on cluster, build all the ABINIT external libraries from source with abinit-fallbacks, so that ABINIT can then be built depending on exactly those libraries.

- Only the **compiler, MPI and math library** come from cluster modules. Since abinit-fallbacks uses the libraries already available (section 4), no other module is loaded, in particular no `cray-hdf5`, `cray-netcdf` or EasyBuild library: HDF5, netCDF, LibXC, ELPA, Wannier90, BigDFT, … are all built here. None of the fallbacks uses FFTW.
- Everything runs in a **clean environment**: `env -i`, then `source /etc/profile` (only for the `module` command), then the module lines of the toolchain.
- No **configure argument** is needed, except for the library paths of the toolchain. The compilers, MPI and linear algebra are detected automatically.

## 2. The six toolchains

Here we take six toolchains of Lucia as an example. The modules, compilers, MPI and math library are:

| Name | Modules | Compilers | MPI | Math library |
|---|---|---|---|---|
| `cray_cray` | `Cray/24.07` `PrgEnv-cray/8.4.0` | CCE 18.0.0 | Cray MPICH 8.1.30 | Cray LibSci 24.07 |
| `cray_gnu` | `Cray/24.07` `PrgEnv-gnu/8.4.0` | GCC 13.3 | Cray MPICH 8.1.30 | Cray LibSci 24.07 |
| `cray_intel` | `Cray/24.07` `PrgEnv-intel/8.4.0` | icx 2022.2 + ifort 2021.7 | Cray MPICH 8.1.30 | Cray LibSci 24.07 |
| `cray_aocc` | `Cray/24.07` `PrgEnv-aocc/8.4.0` | AOCC 4.1 (clang 16 + classic flang) | Cray MPICH 8.1.30 | Cray LibSci 24.07 |
| `eb_intel` | `EasyBuild/2025a` `intel/2025a` | icx/ifx 2025.1 | Intel MPI 2021.15 | MKL 2025.1 |
| `eb_foss` | `EasyBuild/2025a` `foss/2025a` | GCC 14.2 | OpenMPI 5.0.7 | FlexiBLAS 3.4.5 / OpenBLAS 0.3.29, ScaLAPACK 2.2.2 |

Each build directory has an `env.sh` with exactly these lines, for example:

```bash
module --force purge          # also unloads the sticky Cray/ and EasyBuild/ modules
module load Cray/24.07
module load PrgEnv-gnu/8.4.0
```

## 3. Packages

All packages are installed as **static libraries**, one directory per toolchain (`$FB` in the ABINIT report):

| Package | Version |
|---|---|
| HDF5 (parallel) | 1.14.6 |
| netCDF-C (parallel) | 4.9.3 |
| netCDF-Fortran | 4.6.2 |
| LibXC | 7.0.0 |
| ELPA | 2025.01.001 |
| Wannier90 | 2.0.1.1 |
| XMLF90 | 1.6.3 |
| libPSML | 2.1.0 |
| BigDFT | abinit-1.7.1.34 |
| AtomPAW | 4.2.0.3 (program only) |

## 4. The abinit-fallbacks build system

Compared with upstream abinit-fallbacks, `my_develop` does the following.

- **Compilers**, when `CC`, `CXX` and `FC` are not set:
  - in a Cray PE, the craype wrappers `cc`/`CC`/`ftn`;
  - otherwise the MPI wrappers (`mpiicx`/`mpiicpx`/`mpiifx`, then `mpicc`/`mpicxx`/`mpif90`), then the serial compilers: Intel, then LLVM (`clang`/`clang++` with `flang`), then GNU;
  - configure stops if the Fortran compiler cannot create executables.
- **Linear algebra**, detected as ABINIT does (same `--with-linalg-flavor`), except that the auto mode tries first:
  - Cray LibSci in a Cray PE (`libsci` flavor: the craype wrappers link it implicitly, so no library has to be added);
  - MKL with the Intel compilers, when `MKLROOT` is set;
  - FlexiBLAS, then OpenBLAS, which include LAPACK, with pkg-config or by library name;
  - ScaLAPACK, which includes BLACS. Only the library directory is read from `scalapack.pc`, which requires `mpi.pc` and `blas.pc`, usually missing.

  The internal LINALG fallback is only built when no LAPACK is found.
- **Libraries already available:** LibXC, HDF5, netCDF-C, netCDF-Fortran and ELPA are looked for with pkg-config, and used instead of the fallback when they compile and link with the current compilers. configure reports them with "Install: no".
  - pkg-config runs with `PKG_CONFIG_ALLOW_SYSTEM_CFLAGS=1 PKG_CONFIG_ALLOW_SYSTEM_LIBS=1`. pkgconf otherwise drops the `-I`/`-L` flags of the directories listed in `CPATH` and `LIBRARY_PATH`, which modules set, and gfortran does not look for `.mod` files in `CPATH`.
  - HDF5 and netCDF-C must support parallel I/O if and only if MPI is available. netCDF-C is only taken with an external HDF5, and netCDF-Fortran with an external netCDF-C, so that the whole I/O stack comes from one installation.
  - The default search paths of the compilers are not used. They give empty include flags, with which AtomPAW's configure stops and BigDFT builds its bundled LibXC.
  - `--enable-<pkg>` forces the fallback, `--disable-<pkg>` does without it, and `--with-<pkg>-incs`/`--with-<pkg>-libs` give an external package explicitly, checked the same way. LibXC is checked through its C interface, the only one ABINIT, AtomPAW and BigDFT use.
- **ELPA:**
  - built only with MPI and ScaLAPACK;
  - only the library, not its test programs;
  - the SIMD kernels the build host supports (SSE, AVX, AVX2, AVX-512); `--disable-elpa-simd` gives generic kernels;
  - a one-line patch for Cray Fortran (a non-standard continuation line).
- **netCDF-C:** `--disable-libxml2` (bundled tinyxml2) and `--disable-dap`.
- **LibXC:** `-O0` with the Intel compilers and with GCC (section 7).
- **BigDFT:**
  - BigDFT's configure only tries `-lscalapack`. Without it, BigDFT compiles `src/modules/blacs_fake.f90`, stubs such as `blacs_gridinit` or `pdgemm` that just `stop 'FAKE …'`. In a program linking `libbigdft-1.a`, these stubs **replace the real routines** whenever the real library comes later on the link line. That is the case with the Cray wrappers, which add LibSci at the very end: every ScaLAPACK run of ABINIT would stop in `FAKE BLABS_GRIDINIT`. The fallbacks therefore pass `--with-scalapack=-lsci_<PrgEnv>_mpi` (LibSci) or `-lmkl_scalapack_lp64` (MKL) to BigDFT;
  - `PYTHON=python3`, since its bundled S_GPU stops without a `python` command;
  - with the static netCDF fallbacks, the full dependency list of `nc-config --static --libs`;
  - `-fallow-argument-mismatch` with GCC ≥ 10, `-h ipa1` with CCE 18 (internal compiler error, "Broken module found"), `-O1` with AOCC 4.1 (flang crashes on `src/forces.f90`);
  - not built by default with LLVM Flang, which crashes (`not a typeless expr`) on its allocations such as `f_malloc_ptr((/ 3, nat /),id='wfn%rxyz')`, an idiom used about 1200 times (`--enable-bigdft` still tries; the rule does not apply to the Flang of AOCC).
- **Wannier90:** `-fallow-argument-mismatch` with GCC ≥ 10.
- **XMLF90:** the module directories of all its parts are on the include path, which its examples need with AOCC and other flang-based compilers.
- **AtomPAW:** patched to read the UPF grid parameters within its `CHARACTER(200)` input line (the substrings went up to position 256, which Flang refuses).
- **HDF5:** patched so that `clang++` is not taken for `g++` when HDF5 records the compiler versions. With the LLVM of EasyBuild, the result spans two lines and breaks the Makefiles.
- **Cray Fortran:** `-ef`, to get lower-case module file names (`netcdf.mod`, not `NETCDF.mod`).
- **`abinit-fallbacks-config`:** the right BigDFT path, and no `libatompaw` (AtomPAW is only a program).

## 5. Lucia tests: build and check

`build.sh` in each directory does, from any shell:

```bash
env -i HOME=$HOME USER=$USER LOGNAME=$LOGNAME TERM=$TERM PATH=/usr/bin:/bin \
    bash --noprofile --norc
source /etc/profile
source env.sh
../configure --prefix=$PWD      # compilers and math library are detected automatically
make
make install
./check.sh                      # writes build-summary.txt
```

`check.sh` verifies that:

- every package is installed;
- `bin/abinit-fallbacks-config` returns existing directories;
- no installed program depends on a shared HDF5, netCDF or LibXC library from the cluster.

The tuned options target the build node's CPU: ELPA's SIMD kernels, `-march=native`, and `craype-x86-milan` on Cray. On a different CPU, rebuild; for ELPA, `--disable-elpa-simd` gives generic kernels.

## 6. Workstation tests

### 6.1 Setup

Three EasyBuild 2025b toolchains, each of which also provides HDF5 1.14.6, netCDF 4.9.3 and netCDF-Fortran 4.6.2 (all with parallel I/O), LibXC 7.0.0 and ELPA 2025.06.002, but not Wannier90, XMLF90, libPSML, BigDFT or AtomPAW:

| Name | Module | Compilers | MPI | Math library |
|---|---|---|---|---|
| `foss` | `foss/2025b` | GCC 14.3 | OpenMPI 5.0.8 | FlexiBLAS 3.4.5 / OpenBLAS 0.3.30, ScaLAPACK 2.2.2 |
| `intel` | `intel/2025b` | icx/ifx 2025.2 | Intel MPI 2021.16 | MKL 2025.2 |
| `lfoss` | `lfoss/2025b` | LLVM 20.1.8 (clang + flang) | OpenMPI 5.0.8 | FlexiBLAS 3.4.5 / OpenBLAS 0.3.30, ScaLAPACK 2.2.2 |

Three scenarios differ only in the modules loaded besides the toolchain:

| Scenario | Modules loaded besides the toolchain |
|---|---|
| `fb` | none |
| `mix` | `HDF5/1.14.6 netCDF/4.9.3 netCDF-Fortran/4.6.2 libxc/7.0.0 ELPA/2025.06.002` |
| `partial` (`foss` only) | `HDF5/1.14.6 libxc/7.0.0` |

Every build runs in a clean environment, with **no configure argument**:

```bash
env -i HOME=$HOME USER=$USER LOGNAME=$LOGNAME TERM=$TERM PATH=/usr/bin:/bin \
    bash --noprofile --norc
source /etc/profile
module load foss/2025b
module load HDF5/1.14.6 netCDF/4.9.3 netCDF-Fortran/4.6.2 libxc/7.0.0 ELPA/2025.06.002   # mix only
mkdir $FB && cd $FB
<abinit-fallbacks>/configure --prefix=$FB
make && make install
```

### 6.2 What configure selects

- **Compilers:** the MPI wrappers, `mpicc`/`mpicxx`/`mpif90` for `foss` (GNU 14.3) and `lfoss` (LLVM 20.1), `mpiicx`/`mpiicpx`/`mpiifx` for `intel`.
- **Linear algebra:** `flexiblas+scalapack_pkg` (`-lscalapack -lflexiblas`) for `foss` and `lfoss`, `mkl` (`-qmkl=cluster`) for `intel`. The internal LINALG fallback is never built.
- **External packages:** each one is looked for with pkg-config (`hdf5_hl`, `libxc`, `elpa`, `netcdf`, `netcdf-fortran`) and used instead of the fallback when it works with the current compilers. The summary of configure marks it with "Install: no".

| Build | Built by the fallbacks | Taken from EasyBuild |
|---|---|---|
| `foss` fb | all ten packages | none |
| `foss` mix | Wannier90, XMLF90, libPSML, BigDFT, AtomPAW | HDF5, netCDF, netCDF-Fortran, LibXC, ELPA |
| `foss` partial | netCDF, netCDF-Fortran, ELPA, Wannier90, XMLF90, libPSML, BigDFT, AtomPAW | HDF5, LibXC |
| `intel` fb | all ten packages | none |
| `intel` mix | Wannier90, XMLF90, libPSML, BigDFT, AtomPAW | HDF5, netCDF, netCDF-Fortran, LibXC, ELPA |
| `lfoss` fb | all but BigDFT | none |
| `lfoss` mix | Wannier90, XMLF90, libPSML, AtomPAW | HDF5, netCDF, netCDF-Fortran, LibXC, ELPA |

Notes:

- In `foss` partial, netCDF-C is built against the HDF5 of EasyBuild, so the whole I/O stack uses one HDF5: its `libnetcdf.settings` lists `-L<EasyBuild HDF5>/lib -lhdf5_hl -lhdf5`, with NC-4 parallel support.
- BigDFT is not built with LLVM Flang, which crashes on its source code (`LLVM Flang cannot compile BigDFT => disabling the BigDFT fallback`).
- ABINIT built against each of these seven installations, and passed its smoke tests: see section 6 of the [ABINIT build system](/post/abinit-build-system.html).

### 6.3 Environments without a complete toolchain

The same command, still with no configure argument, also works when only part of a toolchain is loaded. configure was run in each of the following environments, and `make install` in those marked OK (3 to 5 min each). The modules are those of the hierarchical EasyBuild tree, e.g. `module load GCC/14.3.0 OpenBLAS/0.3.30`.

Compilers and linear algebra only:

| Modules | Compilers | Linear algebra | Not built | `make install` |
|---|---|---|---|---|
| none (system GCC 13.3) | `gcc` / `gfortran` | internal LINALG | ELPA | – |
| `GCC` | `gcc` / `gfortran` | internal LINALG | ELPA | OK |
| `intel-compilers` | `icx` / `ifx` | internal LINALG | ELPA | OK |
| `llvm-compilers` | `clang` / `flang` | internal LINALG | ELPA, BigDFT | OK |
| `gompi` | `mpicc` / `mpif90` | internal LINALG | ELPA | OK |
| `iimpi` | `mpiicx` / `mpiifx` | internal LINALG | ELPA | OK |
| `lompi` | `mpicc` / `mpif90` (clang/flang) | internal LINALG | ELPA, BigDFT | OK |
| `GCC` + `OpenBLAS` | `gcc` / `gfortran` | `openblas_pkg` | ELPA | OK |
| `GCC` + `FlexiBLAS` | `gcc` / `gfortran` | `flexiblas` | ELPA | – |
| `GCC` + `BLIS` or `AOCL-BLAS` | `gcc` / `gfortran` | internal LINALG | ELPA | – |
| `GCC` + `imkl` | `gcc` / `gfortran` | `mkl` (`-lmkl_gf_lp64 -lmkl_sequential -lmkl_core`) | ELPA | – |
| `intel-compilers` + `imkl` | `icx` / `ifx` | `mkl` (`-qmkl`) | ELPA | – |
| `gompi` + `FlexiBLAS` | `mpicc` / `mpif90` | `flexiblas` | ELPA | OK |
| `llvm-compilers` + `FlexiBLAS` | `clang` / `flang` | `flexiblas` | ELPA, BigDFT | – |

A complete toolchain plus one library:

| Modules | Taken from EasyBuild | Not built | `make install` |
|---|---|---|---|
| `foss` + `ELPA` | ELPA | – | – |
| `intel` + `ELPA` | ELPA | – | – |
| `lfoss` + `ELPA` | ELPA | BigDFT | – |
| `foss` + `netCDF` | HDF5, netCDF | – | OK |
| `lfoss` + `netCDF-Fortran` | HDF5, netCDF, netCDF-Fortran | BigDFT | – |

Notes:

- "Internal LINALG" is the BLAS/LAPACK of the LINALG fallback. It is built when no LAPACK is found, also with BLIS or AOCL-BLAS, which only provide BLAS.
- ELPA needs MPI and ScaLAPACK: configure disables it without them (`MPI is not available` or `ScaLAPACK is not available => disabling the ELPA fallback`).
- With only `llvm-compilers`, configure takes `clang`/`flang`, not the `gcc`/`gfortran` of the GCCcore module that LLVM depends on.
- The `nvhpc` module sets `CC=nvc` and `FC=nvfortran`, and configure honours them: the build has no MPI, although the HPC-X `mpicc`/`mpif90` are in `PATH` (section 7).
- In `foss` + `netCDF`, the netCDF module also loads HDF5, and netCDF-Fortran is built against both.

### 6.4 Without pkg-config

LibXC, HDF5, netCDF, netCDF-Fortran and ELPA are only looked for with pkg-config. Without it (tested with `PKG_CONFIG=no` and the `mix` modules of `foss` and `intel`), configure warns:

```
configure: WARNING: pkg-config not found: the fallbacks will be built even for
                  the packages already available, except linear algebra
```

and builds all the fallbacks but LINALG. The linear algebra is still found from the library names only: `-lscalapack -lflexiblas` for `foss`, `-qmkl=cluster` for `intel`.

## 7. Open points

- **Fake ScaLAPACK in BigDFT** (section 4). BigDFT's configure could look for ScaLAPACK under other names, so that its stubs are never built when a real library is available.
- **LibXC at `-O0` with GCC.** The LibXC tricks give `-O0` to the GNU compilers, under a comment and a message about the Cray compiler (`compilation with cray gets stuck`), and nothing to CCE. The LibXC fallback is thus not optimised with GCC (`CFLAGS_LIBXC = -g -O2 -O0` with `foss`). Which compiler really needs it remains to be checked.
- **C compiler type of `mpiicx`.** configure reports `checking for compiler type of 'mpiicx'... gnu`, with no version. The oneAPI check only reads the first line of `mpiicx -V`, which is the banner of the Intel MPI wrapper (`mpiicx for the Intel(R) MPI Library 2021.16`), and the GNU check matches afterwards. The tricks use the Fortran compiler (`intel 2025.2`) and MKL is selected, so this had no effect on the builds of section 6.
- **NVHPC and full GPU offload.** Neither is supported nor tested for now. The `nvhpc/26.9` row of section 6.3 only records what configure selects today with that module: since the module exports `CC=nvc` and `FC=nvfortran`, the build has no MPI, although the HPC-X wrappers are available.
- **BLAS-only libraries.** With `BLIS/2.0` or `AOCL-BLAS/5.1` alone, there is no LAPACK, and the internal LINALG fallback is built instead of using them for BLAS.
- **BigDFT with LLVM Flang.** To check again with newer Flang versions, with `--enable-bigdft`.
- **XMLF90 download.** Its first URL (GitLab) now serves a repackaged tarball, with the same 211 files but another MD5. configure warns (`xmlf90-1.6.3 tarball MD5 check failed`) and downloads it again from forge.abinit.org.
