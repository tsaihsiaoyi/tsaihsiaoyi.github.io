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
> - Lucia, six toolchains (four Cray PE, two EasyBuild 2025a): configure succeeds and `config.h` defines every required macro (section 5). The full builds and the run-time tests remain to be done: section 7 lists them.

Versions:

- ABINIT 10.9.3 (development), branch `cray_autodetect`, commit `6cd90d43b2`: the eight commits of section 3 on top of upstream develop `a0d07ec4c5`.
- External libraries:
  - Lucia: abinit-fallbacks, one static installation per toolchain (`$FB` below). Commit `6266085` for the configure test of section 5, `b6aa2ef` for the tests of section 7;
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
- **BLACS in ScaLAPACK** (same commit). The `netlib` flavor looked for BLACS only as `-lblacs -lblacsCinit -lblacsF77init`, the layout of ScaLAPACK 1.x. ScaLAPACK ≥ 2.0 includes BLACS. When these libraries do not work, BLACS is now looked for in the ScaLAPACK libraries. Before, they also stayed in `LIBS` after the failed check, so the ScaLAPACK check failed as well. With these two changes, an OpenBLAS + ScaLAPACK stack such as EasyBuild's `foss` needs no `--with-linalg-flavor` (sections 5.2 and 6).
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

- the four Cray toolchains get LibSci through the craype wrappers (section 3.1), and FFTW3 from the `cray-fftw` module (section 5.3);
- `eb_intel` gets MKL, with its DFTI FFT;
- `eb_foss` gets OpenBLAS with ScaLAPACK (section 3.4), and EasyBuild's FFTW3 through pkg-config.

# 5. Results on Lucia

## 5.1 What `check.sh` verifies

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
8. On Cray only, FFTW comes from the `cray-fftw` module:
   - the FFT flavor is `fftw3`;
   - the FFTW3 flags point to `$FFTW_ROOT`;
   - `abinit -b` lists `HAVE_FFTW3`;
   - `ldd abinit` resolves `libfftw3*` to a `cray-fftw` file.

## 5.2 Configure test

On 2026-11-02 the six toolchains were checked with commit `6cd90d43b2`, against the fallbacks at `6266085`. This test stops after configure:

- configure runs with the command of section 4, with no other argument for any toolchain;
- checks 1–5 and 8 of section 5.1 are done on the configure results: no `.ac9` file, the compilers, MPI and MPI-IO, linear algebra with ScaLAPACK and ELPA, every library path in `$FB`, and FFTW from `cray-fftw` on Cray;
- `config.h` defines `HAVE_MPI HAVE_MPI_IO HAVE_HDF5 HAVE_HDF5_MPI HAVE_NETCDF HAVE_NETCDF_MPI HAVE_NETCDF_FORTRAN HAVE_NETCDF_FORTRAN_MPI HAVE_LIBXC HAVE_LINALG_SCALAPACK HAVE_LINALG_ELPA HAVE_WANNIER90 HAVE_LIBPSML HAVE_XMLF90 HAVE_BIGDFT`, plus `HAVE_FFTW3` on Cray.

Without `make`, the configures ran on a login node. All six pass:

| Toolchain | Extra arguments | Linear algebra (configure) | FFT |
|---|---|---|---|
| `cray_cray`, `cray_gnu`, `cray_intel`, `cray_aocc` | none | `elpa+libsci` | FFTW3 (cray-fftw), + `HAVE_FFTW3_MPI` |
| `eb_intel` | none | `elpa+mkl` | DFTI |
| `eb_foss` | none | `elpa+netlib+openblas` | FFTW3 (EasyBuild) |

- **`eb_foss` without `--with-linalg-flavor`.** BLACS is first looked for as `-lblacs …`, which ScaLAPACK 2.2.2 does not provide, then inside ScaLAPACK (`7f3fa29eb3`). The libraries are `-lelpa -lscalapack -lopenblas -lpthread`:

  ```
  checking for the serial linear algebra detection sequence... openblas atlas netlib openblas_pkg
  checking for the MPI linear algebra detection sequence... netlib
  checking for BLACS support in the specified libraries... no
  checking for BLACS support in the specified libraries... yes
  checking for the actual linear algebra flavor... elpa+netlib+openblas
  ```

- **`cray_aocc` with the LLVM commits.**
  - LibSci is still tried first. The serial sequence is `libsci openblas atlas netlib openblas_pkg`, with OpenBLAS added for LLVM by `a84e3a012d`.
  - configure prints `adding the linear algebra Fortran flags to FCFLAGS for Flang` (`f7fb136e1c`).
  - The optimization flags are `-O2 -march=native` (`77acf5425e`).

## 5.3 FFTW on Cray: `cray-fftw`

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

`check.sh` prints both resolutions.

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

# 7. Tests to run on Lucia

Section 5 only ran configure with `6cd90d43b2`. The tests below complete the check of this commit on Lucia, for the six toolchains of section 2: full builds (T1), a few run-time smoke tests (T2), and the FFTW library loaded at run time on Cray (T3). They are not a run of the test suite. They follow the rules of section 1: clean environment, no argument besides the library paths, heavy work in Slurm jobs on the debug partition.

## 7.1 Setup

- **ABINIT.** A source tree (`$ABI` below) on branch `cray_autodetect`, with `git rev-parse --short=10 HEAD` giving `6cd90d43b2` and a clean `git status`. If the branch has moved, test its new head and report the commit. Regenerate the build system once, before any build: `config/scripts/makemake` (with `--no-precommit-hooks` if the Git hook is not installed there). No `git pull` until all builds are done (section 8).
- **Fallbacks.** `FB=~/program/abinit-fallbacks/_build_<toolchain>`: the six installations of commit `b6aa2ef`, for which the fallbacks' `check.sh` reports `RESULT: OK` (see the fallbacks report). Report the commit with the results.
- **Build directories.** One new, empty directory per toolchain, `$ABI/_build_<toolchain>`: these must be clean builds, so an existing directory is wiped or renamed first. It holds `env.sh` (section 2) and `check.sh` (section 5.1: the `check.sh` already on Lucia does checks 1–8).
- **Python.** `runtests.py` stops with Python < 3.10, and the default `python3` of RHEL 8 is 3.6. Call a Python ≥ 3.10 by its absolute path (for example from a virtual environment in `$HOME`), without loading a Python module into the toolchain environment.
- **Slurm variables.** `env -i` also removes the `SLURM_*` variables, without which `srun` inside a job asks for a new allocation. The test jobs save them before cleaning the environment, and source them after `env.sh` (T2).

## 7.2 T1: full builds

One Slurm job per toolchain, on a whole debug node. CCE is the slowest compiler: give `cray_cray` the longest time limit. In each build directory, `build.sh`:

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

Expected:

- `make` exits with 0, and `check.sh` reports `RESULT: OK` (checks 1–8 of section 5.1);
- configure selects the linear algebra and FFT of section 5.2;
- `cray_aocc`: this is the first full build with the commits of section 3.4 and classic flang. `config.log` shows `-march=native` in the optimization flags and `adding the linear algebra Fortran flags to FCFLAGS for Flang`, and every directory compiles;
- `cray_cray`: no `ftn-1279` error in `make.out` (section 3.3), and `abinit` links with BigDFT (`libyaml` order, section 3.1);
- `eb_foss`: `elpa+netlib+openblas`, and `ldd abinit` shows EasyBuild's `libopenblas` and `libscalapack`, never `/usr/lib64/libopenblas*`.

Record, for each toolchain, the linear algebra and FFT flavors, and the `ldd abinit` lines of the linear algebra and of FFTW.

## 7.3 T2: smoke tests

After T1, one Slurm job per toolchain with 4 tasks:

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
MPI_ARGS=--mpi=cray_shasta           # eb_intel: --mpi=pmi2, eb_foss: --mpi=pmix (section 8)
export LD_LIBRARY_PATH=$CRAY_LD_LIBRARY_PATH:$LD_LIBRARY_PATH   # Cray only: Milan FFTW (section 5.3)
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

With `--use-srun`, the serial tests also run with `srun -n 1`. Expected: in `tests_seq/results.json` and `tests_mpi/results.json`, every test is `succeeded` or `passed`. A `skipped` test means that a CPP option is missing (`HAVE_BIGDFT`, for example), so it counts as a failure.

## 7.4 T3: FFTW at run time on Cray

For each of the four Cray builds, after T2:

- `ldd src/98_main/abinit | grep fftw` with and without the `LD_LIBRARY_PATH` line of `tests.sh`: give the directory of each `libfftw3*` (`x86_milan` or `x86_rome`);
- rerun `v1:01` and `paral:24` without that line, in another work directory (`-w tests_rome`): they must pass too;
- optional, `cray_gnu` only: a second build directory whose `build.sh` exports `CRAY_ADD_RPATH=yes` before configure. Check that `ldd abinit` gives the `x86_milan` libraries without `LD_LIBRARY_PATH`, and that `v1:01` passes (section 9).

## 7.5 Results to report

For each toolchain:

- the ABINIT and fallbacks commits;
- whether `make` succeeded and `check.sh` reported `RESULT: OK`, and the failed checks otherwise;
- the linear algebra and FFT flavors, and the `ldd` lines of T1;
- the status of each test of T2 and T3.

For a failure, give the first error of `make.out` or of the test log, and the path of the `.fldiff` file. Never update a reference file to make a test pass. These results then replace the configure test of section 5.2, and the tests done leave this section.

# 8. Pitfalls worth knowing

- **Automake rebuild rules.** ABINIT's Makefiles have no "maintainer mode". If a file of `config/m4` is newer than `configure`, every `make` reruns aclocal/autoconf/automake *in the shared source tree*, and concurrent builds collide (`autom4te: cannot rename …`).
  - `aclocal` does not rewrite `aclocal.m4` when only the content of an included m4 file changes. So after `./autogen.sh`, touch `aclocal.m4`, `config.h.in`, `configure` and the `Makefile.in` files, in that order.
- **No `git pull` while builds run.** All the build directories share one source tree:
  - a running `make` that sees a newer `configure.ac` reruns `aclocal` and `autoconf` in the source tree, then `config.status --recheck`, i.e. configure again in the middle of the build. The binary then mixes objects compiled with two different `config.h`;
  - a build that is in configure when `configure` is rewritten stops with `syntax error near unexpected token`.

  After a pull, regenerate the build system with `config/scripts/makemake`, because changes to `config/optim` only reach `configure` that way, not through the rebuild rules. Then rebuild from scratch.
- **MPI launch inside a Slurm job** (Slurm 25.11, no default MPI plugin). Each library needs its own plugin, or the ranks start as separate singletons:
  - Cray MPICH: `srun --mpi=cray_shasta`;
  - Intel MPI: `srun --mpi=pmi2` with `I_MPI_PMI_LIBRARY=/usr/lib64/libpmi2.so`;
  - OpenMPI 5: `srun --mpi=pmix`.

# 9. Open points

- **No linear algebra is only a warning.** When nothing works, configure prints `linear algebra does not work` and goes on (policy `warn`), so the error only appears when the executables are linked, after the whole build.
- **`FCFLAGS_HINTS`** cannot override the vendor hints. `configure.ac` applies the user's value (`ABI_ENV_RECALL`, line 136) *before* computing the vendor hints (`ABI_FC_HINTS`, line 403), which overwrite it. With the AOCC changes of section 3.2, Lucia does not need it.
- **Upstream report** of AOCC flang's preprocessor bug.
- **FFTW variant at run time on Cray.** By default the loader picks the `x86_rome` build of `cray-fftw` (section 5.3). It would be cleaner to embed the module's library path in the binary, for example by linking with `CRAY_ADD_RPATH=yes` (optional part of T3).
- **FFTW3 threads.** OpenMP is off in these builds, so ABINIT uses `fftw3`, not `fftw3-threads`, although `cray-fftw` provides the threaded libraries.
