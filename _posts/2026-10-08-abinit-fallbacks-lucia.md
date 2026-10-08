---
title: ABINIT + abinit-fallbacks on Lucia (six toolchains)
date: '2026-10-08 16:00:00'
permalink: /post/abinit-fallbacks-lucia.html
layout: post
published: true
---



# ABINIT + abinit-fallbacks on Lucia (six toolchains)

> **Status, 2026-10-08.** ABINIT builds on all six toolchains of Lucia, against external libraries built from source with abinit-fallbacks. That includes BigDFT, AOCC, and every build check. With the changes of section 5, the four Cray toolchains need **no configure argument** besides the library paths. The test suite was not part of this work.

Versions:

- ABINIT 10.9.3 (development):
  - branch `fix_hdf5`, commit `11cc7b0a57` (sections 4.1–4.3);
  - then branch `cray_autodetect` on top of it, commits `0d793ca5da`, `41f1b841a2` and `892bc37de3` (section 5; not published yet).
- abinit-fallbacks, fork [tsaihsiaoyi/abinit-fallbacks](https://github.com/tsaihsiaoyi/abinit-fallbacks):
  - branch `my_develop`, commit `3ddec17`;
  - then branch `bigdft_xc_fix` on top of it, commits `46c9322`, `f36663b` and `6266085` (section 3.3; not published yet).
- Cluster: Lucia (Cenaero), debug nodes with 2 × AMD EPYC 7763 (Zen 3, AVX2, no AVX-512), RHEL 8, Slurm 25.11.

# 1. Goal and rules

For each of the six compiler toolchains available on Lucia:

1. build all the ABINIT external libraries from source with abinit-fallbacks;
2. build ABINIT against exactly those libraries, with the **smallest possible set of configure arguments**.

The rules were the same for both steps:

- Only the **compiler, MPI and math library** come from cluster modules. HDF5, netCDF, LibXC, ELPA, Wannier90, BigDFT, … all come from the fallbacks, never from `cray-hdf5`, `cray-netcdf` or EasyBuild modules.
- Everything runs in a **clean environment**: `env -i`, then `source /etc/profile` (only for the `module` command), then the module lines of the toolchain. My `~/.bashrc` activates a conda environment, and two hidden dependencies came from it before I switched to `env -i`:
  - netCDF-C picked up conda's `xml2-config`;
  - BigDFT needed conda's `python`.
- Every configure argument beyond the library paths must be justified by a failing attempt without it. Each attempt's `config.log` is kept.
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

## 3.3 BigDFT: three more fixes, needed to link it into ABINIT

These are in branch `bigdft_xc_fix`. Each one surfaced only once ABINIT was configured with `--with-bigdft`:

1. **Private routines used from outside** (patch `bigdft-abinit-1.7.1.33-0001.patch`, commit `46c9322`). BigDFT's `src/modules/xc.f90` calls `abi_libxc_functionals_getrefs` and `abi_libxc_functionals_constants_load`, but the bundled libABINIT declares them `private`. The compilers then treat the calls as calls to external procedures that don't exist, so **every** program linking `libbigdft-1.a` fails:

   ```
   undefined reference to `abi_libxc_functionals_constants_load_'
   ```

   The patch makes the two routines public.

2. **Missing module files** (patch `-0002`, commit `f36663b`). BigDFT installs its own `.mod` files but not those of the bundled libABINIT (`abi_libxc_functionals.mod`, …). Classic flang (AOCC) needs every module used indirectly, so the patch adds an install rule for them.

3. **Fake ScaLAPACK** (commit `6266085`). BigDFT's configure only tries `-lscalapack`. Without it, BigDFT compiles `src/modules/blacs_fake.f90`, stubs such as `blacs_gridinit` or `pdgemm` that just `stop 'FAKE …'`. In a program linking `libbigdft-1.a`, these stubs **replace the real routines** whenever the real library comes later on the link line. That is the case with the Cray wrappers, which add LibSci at the very end. Every ScaLAPACK run of ABINIT then stopped in `FAKE BLABS_GRIDINIT`. The fallbacks now pass `--with-scalapack=-lsci_<PrgEnv>_mpi` (LibSci) or `-lmkl_scalapack_lp64` (MKL) to BigDFT.

## 3.4 Build

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

## 3.5 Results

All six builds pass (`RESULT: OK`). They were run as Slurm jobs with 16 cores each, from commit `6266085`:

| Toolchain | Compilers detected | Math library detected | Time |
|---|---|---|---|
| `cray_cray` | `cc` / `CC` / `ftn` (CCE 18.0) | `libsci` | 25 min |
| `cray_gnu` | `cc` / `CC` / `ftn` (GCC 13.3) | `libsci` | 8 min |
| `cray_intel` | `cc` / `CC` / `ftn` (ifort 2021.7) | `libsci` | 14 min |
| `cray_aocc` | `cc` / `CC` / `ftn` (AOCC 4.1) | `libsci` | 16 min |
| `eb_intel` | `mpiicx` / `mpiicpx` / `mpiifx` (2025.1) | `mkl` (`-qmkl=cluster`) | 13 min |
| `eb_foss` | `mpicc` / `mpicxx` / `mpif90` (GCC 14.2) | OpenBLAS (pkg-config) + `-lscalapack` | 12 min |

The tuned options target the build node's CPU: ELPA's SIMD kernels, `-march=native`, and `craype-x86-milan` on Cray. On a different CPU, rebuild; for ELPA, `--disable-elpa-simd` gives generic kernels.

# 4. Step 2: ABINIT `fix_hdf5` as it is

## 4.1 Baseline configure command

The same for every toolchain. It contains only the libraries to use, and no `--prefix` (ABINIT is built in place):

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

## 4.2 Minimal extra arguments needed by `fix_hdf5`

| Toolchain | Extra arguments | Configure attempts |
|---|---|---|
| `eb_intel` | none | 1 |
| `eb_foss` | `--with-linalg-flavor=easybuild+elpa` | 2 |
| `cray_gnu` | `CC=cc CXX=CC FC=ftn LINALG_LIBS="-lsci_gnu_mpi -lsci_gnu"` | 3 (+1 experiment) |
| `cray_cray` | `CC=cc CXX=CC FC=ftn LINALG_LIBS="-lsci_cray_mpi -lsci_cray"` | 3 |
| `cray_intel` | `CC=cc CXX=CC FC=ftn LINALG_LIBS="-lsci_intel_mpi -lsci_intel"` | 3 |
| `cray_aocc` | as `cray_*`, plus `FCFLAGS="-g -Mextend -Qunused-arguments -I$FB/elpa/default/include/elpa-2025.01.001/modules"`, plus 3 source fixes | 5 |

Why each argument was needed:

**`CC=cc CXX=CC FC=ftn` (Cray).** Without them, ABINIT searches `PATH` for `mpiicx mpiicc mpicc`, `mpiicpx mpiicpc mpic++ mpicxx` and `mpiifx mpiifort mpifort mpif90 mpif95` (`config/m4/sd_arch_mpi.m4`). In every PrgEnv it therefore picks cray-mpich's own `mpicc`/`mpic++`/`mpifort`, not the craype wrappers. Those wrappers do not link LibSci and do not add the `craype-x86-milan` target options, with these consequences:

- `cray_gnu`: links the **system** `/usr/lib64/libopenblas.so` (RHEL's OpenBLAS 0.3.15), and LAPACK is still not found;
- `cray_cray`: only the `netlib` flavor is tried, `-lblas` does not exist, and there is no linear algebra at all;
- `cray_intel`: MKL is selected (`-qmkl=cluster`) instead of LibSci: for Intel compilers ABINIT always tries MKL first, and PrgEnv-intel ships one;
- `cray_aocc`: configure stops at the Fortran checks (see below).

**`LINALG_LIBS="-lsci_<x>_mpi -lsci_<x>"` (Cray).** ABINIT had no LibSci flavor: an unknown name fails with `no library settings for linear algebra flavor`. Even with the wrappers, its auto-detection still picks system OpenBLAS (gnu), finds nothing (cray, aocc) or uses MKL (intel). Setting any `LINALG_*` variable switches ABINIT to "verify only" mode, and `--with-elpa` keeps working in that mode.

`LINALG_LIBS` alone is not enough. I checked this on `cray_gnu`: without the wrappers, cray-mpich's `mpifort` cannot find LibSci at all:

```
ld: cannot find -lsci_gnu_mpi: No such file or directory
```

**`--with-linalg-flavor=easybuild+elpa` (eb_foss).** The automatic detection finds EasyBuild's OpenBLAS (through pkg-config) but no ScaLAPACK. Its MPI flavor `netlib` hard-codes `-lblacs -lblacsCinit -lblacsF77init`, which ScaLAPACK 2.2.2 does not have. The `easybuild` flavor uses `-lopenblas -lscalapack`. `+elpa` is needed because an explicit flavor no longer adds the ELPA check by itself (`sd_math_linalg.m4`).

**`FCFLAGS=...` (AOCC).** There are two problems:

- AOCC's `ftn --version` prints `AMD clang version 16.0.3 (CLANG: AOCC_4.1.0...)`. That matches ABINIT's LLVM-flang test, which runs before its AOCC test, so ABINIT applies the new-flang hints `-ffixed-line-length=132 -Qunused-arguments`. Classic flang rejects the first one (`clang-16: error: unknown argument: '-ffixed-line-length=132'`), and configure stops with "cannot compile a simple Fortran program". `-Mextend` is the classic-flang equivalent.
- Classic flang needs module files that are only used *indirectly* (`elpa_api.mod`), but ABINIT passes the ELPA module directory only to the directories that use linear algebra directly. Hence the extra `-I`.

The intended override, `FCFLAGS_HINTS`, does not work. `configure.ac` applies the user's value (`ABI_ENV_RECALL`, line 136) *before* computing the vendor hints (`ABI_FC_HINTS`, line 403), which overwrite it. So the full `FCFLAGS` had to be given.

## 4.3 AOCC 4.1: three source-level problems

Even with the right flags, `make` stopped in ABINIT's own sources:

| File | Problem |
|---|---|
| `shared/libpaw/src/m_paw_atom_solve.F90:7507` | `type(logical)` (Fortran 2008 `TYPE(intrinsic-type)`) is not supported by classic flang: `F90-S-0155-Derived type has not been declared - logical` |
| `src/62_ctqmc/m_CtqmcInterface.F90:148` | Bug in flang's integrated preprocessor: the nested macro `MALLOC` → `ABI_MALLOC` comes out as `C(this%Hybrid_chains,(this%num_chains))` |
| `src/48_diago/m_chebfi2.F90:1386,1663,2029` | Same preprocessor bug inside `ABI_MALLOC_IFNOT` |

The preprocessor bug depends on the data: the same macros expand correctly in hundreds of other places, and in small test files. The fixes are equivalent rewrites (commit `0d793ca5da`):

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

# 5. Branch `cray_autodetect`: no extra arguments on Cray, with BigDFT

## 5.1 Changes to ABINIT's build system

Commit `41f1b841a2` makes configure do what the arguments of section 4.2 did:

- **Cray PE compilers** (`config/m4/arch-mpi.m4`). When `CRAYPE_VERSION` and `PE_ENV` are set and `cc`/`CC`/`ftn` exist, configure uses the craype wrappers unless `CC`/`CXX`/`FC` are given:

  ```
  checking for a Cray Programming Environment... yes (PrgEnv: GNU)
  configure: using the Cray compiler wrappers: CC=cc CXX=CC FC=ftn
  ```

- **`libsci` linear-algebra flavor** (`sd_math_linalg*.m4`), tried first on a Cray PE.
  - With the wrappers, no library has to be named: BLAS, LAPACK, BLACS and ScaLAPACK are all "none required".
  - With other compilers (e.g. `FC=mpifort`), the `libsci_<PrgEnv>` and `libsci_<PrgEnv>_mpi` libraries are linked explicitly, but only when they match the compiler in use.
- **AOCC** (`lang-fortran.m4`, `configure.ac`). AOCC flang is still treated as LLVM, because it needs the LLVM settings. But:
  - `-ffixed-line-length=132` becomes `-Mextend` in its Fortran hints;
  - the linear-algebra Fortran flags (the ELPA modules) are added to `FCFLAGS` for every directory.
- **BigDFT link order** (`sd_bigdft.m4`). `libyaml.a` is now passed as `-Wl,<path>/libyaml.a`. The Cray `ftn` driver moves plain file arguments in front of all `-l` options. That put `libyaml.a` *before* `libbigdft-1.a`, so with CCE every BigDFT link failed with `undefined reference to yaml_parser_initialize`.

Commit `892bc37de3` fixes a real bug that only CCE detected. `m_mklocl_realspace.F90` called BigDFT's `PSolver` with the integer `0` where the interface expects a `type(xc_info)` (`ftn-1279`). The other compilers accepted it, so `PSolver` read an integer as an `xc_info` object. It now passes an object initialised with `xc_init(xc, 0, XC_ABINIT, 1)`.

## 5.2 Configure arguments now

All six toolchains use the library paths of section 4.1, plus `--with-bigdft=$FB/bigdft/default`. Beyond that:

| Toolchain | Extra arguments |
|---|---|
| `cray_cray`, `cray_gnu`, `cray_intel`, `cray_aocc` | **none** |
| `eb_intel` | none |
| `eb_foss` | `--with-linalg-flavor=easybuild+elpa` (as before) |

## 5.3 What "OK" means

Every build ends with a `check.sh` that verifies:

1. No `.ac9` configuration file was read: `not loading options (no config file available)` in `config.log`.
2. The compilers are the expected ones: `cc`/`CC`/`ftn` on Cray, the Intel or OpenMPI wrappers on EasyBuild.
3. MPI works for C, C++ and Fortran, and MPI-IO is enabled.
4. The math library is the cluster's:
   - linear algebra, ScaLAPACK and ELPA are all `yes`;
   - `ldd abinit` shows `libsci_<x>(_mpi)` on Cray, MKL on `eb_intel`, and EasyBuild OpenBLAS/FlexiBLAS + ScaLAPACK on `eb_foss`;
   - never `/usr/lib64/libopenblas*`.
5. Every `-I`/`-L` of HDF5, netCDF, LibXC, ELPA, Wannier90, XMLF90, libPSML and BigDFT points into `$FB`, and none of them (nor `libyaml`) appears as a shared library in `ldd abinit`.
6. `abinit -b` lists `HAVE_MPI HAVE_MPI_IO HAVE_HDF5_MPI HAVE_NETCDF_MPI HAVE_NETCDF_FORTRAN_MPI HAVE_LIBXC HAVE_LINALG_SCALAPACK HAVE_LINALG_ELPA HAVE_WANNIER90 HAVE_LIBPSML HAVE_XMLF90 HAVE_BIGDFT`.
7. `make` exits with 0.

## 5.4 Results

All six builds pass every check (`RESULT: OK`). They used `make -j 64` on one debug node:

| Toolchain | Linear algebra (configure / `ldd`) | FFT | `make` |
|---|---|---|---|
| `cray_cray` | `elpa+libsci` / `libsci_cray(_mpi).so.6` | Goedecker | 32.6 min |
| `cray_gnu` | `elpa+libsci` / `libsci_gnu(_mpi).so.6` | Goedecker | 8.3 min |
| `cray_intel` | `elpa+libsci` / `libsci_intel(_mpi).so.6` (no MKL) | Goedecker | 20.3 min |
| `cray_aocc` | `elpa+libsci` / `libsci_aocc(_mpi).so.6` | Goedecker | 26.1 min |
| `eb_intel` | `elpa+mkl` / MKL 2025.1 (`libmkl_scalapack_lp64`, `libmkl_blacs_intelmpi_lp64`, …) | DFTI | 9.5 min |
| `eb_foss` | `easybuild+elpa` / OpenBLAS 0.3.29, ScaLAPACK 2.2.2, FlexiBLAS 3.4.5 | FFTW3 | 9.1 min |

The times are clean builds of the final build system. The `PSolver` fix (one file) came last: cray_cray was rebuilt from scratch with it, and the other five were brought up to date with an incremental `make`, after which `check.sh` passes again.

No FFTW module is loaded in the Cray environments, so ABINIT uses its internal Goedecker FFT there.

# 6. Pitfalls worth knowing

- **Automake rebuild rules.** ABINIT's Makefiles have no "maintainer mode". If a file of `config/m4` is newer than `configure`, every `make` reruns aclocal/autoconf/automake *in the shared source tree*, and concurrent builds collide (`autom4te: cannot rename …`).
  - `aclocal` does not rewrite `aclocal.m4` when only the content of an included m4 file changes. So after `./autogen.sh`, touch `aclocal.m4`, `config.h.in`, `configure` and the `Makefile.in` files, in that order.
- **MPI launch inside a Slurm job** (Slurm 25.11, no default MPI plugin). Each library needs its own plugin, or the ranks start as separate singletons:
  - Cray MPICH: `srun --mpi=cray_shasta`;
  - Intel MPI: `srun --mpi=pmi2` with `I_MPI_PMI_LIBRARY=/usr/lib64/libpmi2.so`;
  - OpenMPI 5: `srun --mpi=pmix`.

# 7. Reproducing one build

For example `cray_gnu`, from a build directory inside the ABINIT source tree on branch `cray_autodetect`:

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
  --with-bigdft=$FB/bigdft/default
make -j 64
src/98_main/abinit -b      # check the CPP options
```

# 8. Open points

- **Publication.** The branches `cray_autodetect` (ABINIT) and `bigdft_xc_fix` (abinit-fallbacks) are local, not pushed yet.
- **eb_foss** still needs `--with-linalg-flavor=easybuild+elpa`. ABINIT's own detection could look for BLACS inside `libscalapack`, as the fallbacks do.
- **`FCFLAGS_HINTS`** still cannot override the vendor hints (the order problem of section 4.2).
- **Upstream reports.**
  - AOCC flang's preprocessor bug.
  - The three BigDFT 1.7.1.33 problems of section 3.3.
- **FFT on Cray.** FFTW (`cray-fftw`) is not in the module set, so the Cray builds use ABINIT's internal FFT. Features that need FFTW3 or DFTI (e.g. GW-R) are not available there.
