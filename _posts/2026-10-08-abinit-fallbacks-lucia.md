---
title: ABINIT + abinit-fallbacks on Lucia (six toolchains)
date: '2026-10-08 16:00:00'
permalink: /post/abinit-fallbacks-lucia.html
layout: post
published: true
---



# ABINIT + abinit-fallbacks on Lucia (six toolchains)

> **Status report, 2026-10-08.** The external libraries (abinit-fallbacks) build on all six toolchains. ABINIT builds and passes all build checks on five of them; AOCC needs three small source fixes. The test suite has not been run yet.

Versions used:

- ABINIT 10.9.3 (development), branch `fix_hdf5`, commit `11cc7b0a57`
- abinit-fallbacks, fork [tsaihsiaoyi/abinit-fallbacks](https://github.com/tsaihsiaoyi/abinit-fallbacks), branch `my_develop`, commit `3ddec17`
- Cluster: Lucia (Cenaero), debug nodes with 2 × AMD EPYC 7763 (Zen 3, AVX2, no AVX-512), RHEL 8

# 1. Goal and rules

For each of the six compiler toolchains available on Lucia:

1. build all the ABINIT external libraries from source with abinit-fallbacks;
2. build ABINIT against exactly those libraries, with the **smallest possible set of configure arguments**.

The rules were the same for both steps:

- Only the **compiler, MPI and math library** come from cluster modules. HDF5, netCDF, LibXC, ELPA, Wannier90, … all come from the fallbacks, never from `cray-hdf5`, `cray-netcdf` or EasyBuild modules.
- Everything runs in a **clean environment**: `env -i`, then `source /etc/profile` (only for the `module` command), then the module lines of the toolchain. My `~/.bashrc` activates a conda environment, and two hidden dependencies came from it before I switched to `env -i`:
  - netCDF-C picked up conda's `xml2-config`;
  - BigDFT needed conda's `python`.
- Every configure argument beyond the baseline must be justified by a failing attempt without it. Each attempt's `config.log` is kept.
- Heavy work runs as Slurm jobs on the debug partition.

# 2. The six toolchains

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

# 3. Step 1: abinit-fallbacks

## 3.1 Packages

All packages are installed as **static libraries**, one directory per toolchain (`$FB` below):

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
| BigDFT | abinit-1.7.1.33 |
| AtomPAW | 4.2.0.3 (program only) |

## 3.2 Changes to the abinit-fallbacks build system

Upstream abinit-fallbacks did not work out of the box on these toolchains. The changes are in commits `a46c150`…`3ddec17` of my fork:

- **Compilers:**
  - In a Cray PE, use the craype wrappers `cc`/`CC`/`ftn`.
  - Otherwise prefer the MPI wrappers.
- **Linear algebra:**
  - New `libsci` flavor: the craype wrappers link Cray LibSci implicitly, so no library has to be added.
  - BLACS is detected inside `libscalapack` (ScaLAPACK ≥ 2.0 has no separate `libblacs`).
- **Cray Fortran:**
  - `-ef` to get lower-case module file names (`netcdf.mod`, not `NETCDF.mod`).
  - A one-line patch to ELPA (a non-standard continuation line).
- **ELPA:**
  - Build only the library, not its hundreds of test programs.
  - Disabled without MPI.
  - SIMD kernels SSE/AVX/AVX2.
- **netCDF-C:** `--disable-libxml2` (bundled tinyxml2) and `--disable-dap`.
- **BigDFT:**
  - `PYTHON=/usr/bin/python3` (there is no `python` in a clean RHEL 8 environment).
  - Link the full static netCDF/HDF5 dependency list (`nc-config --static --libs`).
  - `-h ipa1` for CCE 18 (internal compiler error, "Broken module found").
  - `-O1` for AOCC 4.1 (flang crashes on `src/forces.f90`).
- **GCC ≥ 10:** `-fallow-argument-mismatch` for BigDFT and Wannier90.
- **XMLF90:** module search paths needed by flang-based compilers.
- **LibXC:** optimisation tweaks for the Intel and Cray compilers.
- **`abinit-fallbacks-config`:** fixed the BigDFT path, and dropped a non-existent `libatompaw`.

## 3.3 Build

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

## 3.4 Results

All six builds pass (`RESULT: OK`). They were run as Slurm jobs with 16 cores each:

| Toolchain | Compilers detected | Math library detected | Time |
|---|---|---|---|
| `cray_cray` | `cc` / `CC` / `ftn` (CCE 18.0) | `libsci` | 24 min |
| `cray_gnu` | `cc` / `CC` / `ftn` (GCC 13.3) | `libsci` | 7 min |
| `cray_intel` | `cc` / `CC` / `ftn` (ifort 2021.7) | `libsci` | 13 min |
| `cray_aocc` | `cc` / `CC` / `ftn` (AOCC 4.1) | `libsci` | 16 min |
| `eb_intel` | `mpiicx` / `mpiicpx` / `mpiifx` (2025.1) | `mkl` (`-qmkl=cluster`) | 12 min |
| `eb_foss` | `mpicc` / `mpicxx` / `mpif90` (GCC 14.2) | OpenBLAS (pkg-config) + `-lscalapack` | 11 min |

The tuned options target the build node's CPU: ELPA's SIMD kernels, `-march=native`, and `craype-x86-milan` on Cray. On a different CPU, rebuild; for ELPA, `--disable-elpa-simd` gives generic kernels.

# 4. Step 2: ABINIT

## 4.1 Baseline configure command

The same for every toolchain. It contains only the libraries to use, and no `--prefix` (ABINIT is built and tested in place):

```bash
../configure \
  --with-hdf5=$FB/hdf5/default \
  --with-netcdf=$FB/netcdf4/default \
  --with-netcdf-fortran=$FB/netcdf4_fortran/default \
  --with-libxc=$FB/libxc/default \
  --with-elpa=$FB/elpa/default \
  --with-wannier90=$FB/wannier90/default \
  --with-xmlf90=$FB/xmlf90/default \
  --with-libpsml=$FB/libpsml/default
```

ABINIT's configure takes only 20–60 s here, so each configure attempt was a short Slurm job, and its logs were archived before the next attempt.

## 4.2 Minimal extra arguments

| Toolchain | Extra arguments | Configure attempts |
|---|---|---|
| `eb_intel` | none | 1 |
| `eb_foss` | `--with-linalg-flavor=easybuild+elpa` | 2 |
| `cray_gnu` | `CC=cc CXX=CC FC=ftn LINALG_LIBS="-lsci_gnu_mpi -lsci_gnu"` | 3 (+1 experiment) |
| `cray_cray` | `CC=cc CXX=CC FC=ftn LINALG_LIBS="-lsci_cray_mpi -lsci_cray"` | 3 |
| `cray_intel` | `CC=cc CXX=CC FC=ftn LINALG_LIBS="-lsci_intel_mpi -lsci_intel"` | 3 |
| `cray_aocc` | `CC=cc CXX=CC FC=ftn LINALG_LIBS="-lsci_aocc_mpi -lsci_aocc"` `FCFLAGS="-g -Mextend -Qunused-arguments -I$FB/elpa/default/include/elpa-2025.01.001/modules"` | 5 |

Why each argument is needed:

**`CC=cc CXX=CC FC=ftn` (Cray).** Without them, ABINIT searches `PATH` for `mpiicx mpiicc mpicc`, `mpiicpx mpiicpc mpic++ mpicxx` and `mpiifx mpiifort mpifort mpif90 mpif95` (`config/m4/sd_arch_mpi.m4`). In every PrgEnv it therefore picks cray-mpich's own `mpicc`/`mpic++`/`mpifort`, not the craype wrappers. Those wrappers do not link LibSci and do not add the `craype-x86-milan` target options, with these consequences:

- `cray_gnu`: links the **system** `/usr/lib64/libopenblas.so` (RHEL's OpenBLAS 0.3.15), and LAPACK is still not found;
- `cray_cray`: only the `netlib` flavor is tried, `-lblas` does not exist, and there is no linear algebra at all;
- `cray_intel`: MKL is selected (`-qmkl=cluster`) instead of LibSci: for Intel compilers ABINIT always tries MKL first, and PrgEnv-intel ships one;
- `cray_aocc`: configure stops at the Fortran checks (see below).

**`LINALG_LIBS="-lsci_<x>_mpi -lsci_<x>"` (Cray).** ABINIT has no LibSci flavor: an unknown name fails with `no library settings for linear algebra flavor`. Even with the wrappers, its auto-detection still picks system OpenBLAS (gnu), finds nothing (cray, aocc) or uses MKL (intel). Setting any `LINALG_*` variable switches ABINIT to "verify only" mode, and `--with-elpa` keeps working in that mode. With LibSci, every check passes: BLAS, LAPACK, BLACS, ScaLAPACK, ELPA, and the ELPA 2017+ Fortran 2008 API.

`LINALG_LIBS` alone is not enough. I checked this on `cray_gnu`: without the wrappers, cray-mpich's `mpifort` cannot find LibSci at all:

```
ld: cannot find -lsci_gnu_mpi: No such file or directory
```

**`--with-linalg-flavor=easybuild+elpa` (eb_foss).** The automatic detection finds EasyBuild's OpenBLAS (through pkg-config) but no ScaLAPACK. Its MPI flavor `netlib` hard-codes `-lblacs -lblacsCinit -lblacsF77init`, which ScaLAPACK 2.2.2 does not have. The `easybuild` flavor uses `-lopenblas -lscalapack`. `+elpa` is needed because an explicit flavor no longer adds the ELPA check by itself (`sd_math_linalg.m4`).

**`FCFLAGS=...` (AOCC).** There are two problems:

- AOCC's `ftn --version` prints `AMD clang version 16.0.3 (CLANG: AOCC_4.1.0...)`. That matches ABINIT's LLVM-flang test, which runs before its AOCC test, so ABINIT applies the new-flang hints `-ffixed-line-length=132 -Qunused-arguments`. Classic flang rejects the first one (`clang-16: error: unknown argument: '-ffixed-line-length=132'`), and configure stops with "cannot compile a simple Fortran program". `-Mextend` is the classic-flang equivalent.
- Classic flang needs module files that are only used *indirectly* (`elpa_api.mod`), but ABINIT passes the ELPA module directory only to the directories that use linear algebra directly. Hence the extra `-I`.

The intended override, `FCFLAGS_HINTS`, does not work. `configure.ac` applies the user's value (`ABI_ENV_RECALL`, line 136) *before* computing the vendor hints (`ABI_FC_HINTS`, line 403), which overwrite it. So the full `FCFLAGS` has to be given.

## 4.3 What "OK" means

Every build ends with a `check.sh` that verifies:

1. No `.ac9` configuration file was read: `not loading options (no config file available)` in `config.log`.
2. The compilers are the expected ones: `cc`/`CC`/`ftn` on Cray, the Intel or OpenMPI wrappers on EasyBuild.
3. MPI works for C, C++ and Fortran, and MPI-IO is enabled.
4. The math library is the cluster's:
   - linear algebra, ScaLAPACK and ELPA are all `yes`;
   - `ldd abinit` shows `libsci_<x>(_mpi)` on Cray, MKL on `eb_intel`, and EasyBuild OpenBLAS/FlexiBLAS + ScaLAPACK on `eb_foss`;
   - never `/usr/lib64/libopenblas*`.
5. Every `-I`/`-L` of HDF5, netCDF, LibXC, ELPA, Wannier90, XMLF90 and libPSML points into `$FB`, and none of them appears as a shared library in `ldd abinit`.
6. `abinit -b` lists `HAVE_MPI HAVE_MPI_IO HAVE_HDF5_MPI HAVE_NETCDF_MPI HAVE_NETCDF_FORTRAN_MPI HAVE_LIBXC HAVE_LINALG_SCALAPACK HAVE_LINALG_ELPA HAVE_WANNIER90 HAVE_LIBPSML HAVE_XMLF90`.
7. `make` exits with 0.

## 4.4 Results

These builds used `make -j 64` on one debug node:

| Toolchain | Configure | `make` | Linear algebra (`ldd`) | FFT | Result |
|---|---|---|---|---|---|
| `eb_intel` | OK | 9.4 min | MKL 2025.1 (`libmkl_scalapack_lp64`, `libmkl_blacs_intelmpi_lp64`, …) | DFTI | **OK** |
| `eb_foss` | OK | 9.0 min | OpenBLAS 0.3.29, ScaLAPACK 2.2.2, FlexiBLAS 3.4.5 | FFTW3 | **OK** |
| `cray_gnu` | OK | 8.2 min | `libsci_gnu_mpi.so.6`, `libsci_gnu.so.6` | Goedecker | **OK** |
| `cray_intel` | OK | 20.2 min | `libsci_intel_mpi.so.6`, `libsci_intel.so.6` (no MKL) | Goedecker | **OK** |
| `cray_cray` | OK | 27.5 min | `libsci_cray_mpi.so.6`, `libsci_cray.so.6` | Goedecker | **OK** |
| `cray_aocc` | OK | fails | (`libsci_aocc` with the patch below) | Goedecker | **needs source fixes** |

No FFTW module is loaded in the Cray environments, so ABINIT falls back to its internal Goedecker FFT there.

# 5. AOCC 4.1: three source-level problems

With the configure arguments above, `make` stops in ABINIT's own sources. Neither problem can be solved with a compiler flag:

| File | Problem |
|---|---|
| `shared/libpaw/src/m_paw_atom_solve.F90:7507` | `type(logical)` (Fortran 2008 `TYPE(intrinsic-type)`) is not supported by classic flang: `F90-S-0155-Derived type has not been declared - logical` |
| `src/62_ctqmc/m_CtqmcInterface.F90:148` | Bug in flang's integrated preprocessor: the nested macro `MALLOC` → `ABI_MALLOC` comes out as `C(this%Hybrid_chains,(this%num_chains))` |
| `src/48_diago/m_chebfi2.F90:1386,1663,2029` | Same preprocessor bug inside `ABI_MALLOC_IFNOT` |

The preprocessor bug depends on the data: the same macros expand correctly in most other places, and in small test files.

To check that nothing else blocks AOCC, I put patched copies of the three files in the build tree only (make's `VPATH` prefers them; the ABINIT source tree was not touched). With them, the whole build completes, and all checks of section 4.3 pass. The changes are equivalent rewrites:

```patch
--- a/shared/libpaw/src/m_paw_atom_solve.F90
+++ b/shared/libpaw/src/m_paw_atom_solve.F90
@@ -7504,7 +7504,7 @@
 SUBROUTINE setcoretail(Grid,coreden,PAW,needvtau)
- type(logical),intent(in) :: needvtau
+ logical,intent(in) :: needvtau
  TYPE(GridInfo), INTENT(IN) :: Grid
--- a/src/62_ctqmc/m_CtqmcInterface.F90
+++ b/src/62_ctqmc/m_CtqmcInterface.F90
@@ -145,7 +145,7 @@
   this%num_chains = num_chains
-  MALLOC(this%Hybrid_chains, (this%num_chains))
+  ABI_MALLOC(this%Hybrid_chains, (this%num_chains))
   this%Hybrid => this%Hybrid_chains(1)
--- a/src/48_diago/m_chebfi2.F90
+++ b/src/48_diago/m_chebfi2.F90
@@ -1383,7 +1383,7 @@
-    ABI_MALLOC_IFNOT(nrowsLinalg,(num_proc))
+    if (.not. allocated(nrowsLinalg)) then; ABI_MALLOC(nrowsLinalg,(num_proc)); endif
     nrowsLinalg_ptr => nrowsLinalg
(same change at lines 1660 and 2026)
```

# 6. Reproducing one build

For example `cray_gnu`, from a build directory inside the ABINIT source tree:

```bash
env -i HOME=$HOME USER=$USER LOGNAME=$LOGNAME TERM=$TERM PATH=/usr/bin:/bin \
    bash --noprofile --norc
source /etc/profile
module --force purge
module load Cray/24.07 PrgEnv-gnu/8.4.0
FB=~/program/abinit-fallbacks/_build_cray_gnu      # the fallbacks built in step 1

../configure \
  --with-hdf5=$FB/hdf5/default --with-netcdf=$FB/netcdf4/default \
  --with-netcdf-fortran=$FB/netcdf4_fortran/default --with-libxc=$FB/libxc/default \
  --with-elpa=$FB/elpa/default --with-wannier90=$FB/wannier90/default \
  --with-xmlf90=$FB/xmlf90/default --with-libpsml=$FB/libpsml/default \
  CC=cc CXX=CC FC=ftn LINALG_LIBS="-lsci_gnu_mpi -lsci_gnu"
make -j 64
src/98_main/abinit -b      # check the CPP options
```

# 7. Open points

- **Tests not run yet.** `tests/runtests.py` cannot run with RHEL 8's `/usr/bin/python3` (3.6.8): `tests/__init__.py` uses `from __future__ import annotations`, which needs Python ≥ 3.7. The system `/usr/bin/python3.12` runs it, but has no numpy or PyYAML, so the YAML-based result checks are degraded. I still have to choose which Python to use for the tests.
- **AOCC:** decide whether to apply the source fixes of section 5, and report the flang preprocessor bug.
- **BigDFT** is built in the fallbacks but not yet enabled in ABINIT (`--with-bigdft`).
- **Possible improvements to ABINIT's build system,** so that the Cray toolchains need no extra arguments:
  - recognise the craype wrappers;
  - add a LibSci flavor (as done in the fallbacks);
  - let `FCFLAGS_HINTS` override the vendor hints;
  - detect AOCC before generic LLVM flang;
  - detect BLACS inside `libscalapack`.
