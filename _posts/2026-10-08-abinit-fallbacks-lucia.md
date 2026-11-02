---
title: ABINIT on Lucia (six toolchains)
date: '2026-10-08 16:00:00'
permalink: /post/abinit-fallbacks-lucia.html
layout: post
published: true
---



# ABINIT on Lucia (six toolchains)

> **Status, 2026-11-02.** ABINIT branch `cray_autodetect` builds on all six toolchains of Lucia, including BigDFT and AOCC, and passes every build check. The four Cray toolchains need **no configure argument** besides the library paths, and use FFTW3 from the `cray-fftw` module without any extra argument (section 5.3). Five later commits (section 3.4) let configure find an OpenBLAS + ScaLAPACK stack such as EasyBuild's `foss`, support LLVM Flang 20, and accept an HDF5 built with CMake. They were tested on a workstation with three EasyBuild 2025b toolchains, with the libraries taken from the fallbacks, from EasyBuild, or from both: none of the seven builds needs an argument besides the paths of the libraries built by the fallbacks (section 6). These commits have not been tested on Lucia yet. The test suite was not part of this work; each workstation build only ran four smoke tests.

Versions:

- ABINIT 10.9.3 (development), branch `cray_autodetect`:
  - Lucia (sections 4–5): commit `892bc37de3`, i.e. commits `0d793ca5da`, `41f1b841a2` and `892bc37de3` on top of `11cc7b0a57` of develop;
  - rebased on 2026-11-02 onto upstream develop (`a0d07ec4c5`), as `8e703f5ffa`, `7087cbaa14` and `15456e01e3`. The changes are the same, plus `libsci` in the `configure --help` text and in `doc/build/config-template.ac9`;
  - workstation (section 6): five more commits, `7f3fa29eb3`, `77acf5425e`, `a84e3a012d`, `f7fb136e1c` and `6cd90d43b2` (section 3.4). All seven builds use `6cd90d43b2`.
- External libraries:
  - Lucia: abinit-fallbacks commit `6266085`, one static installation per toolchain (`$FB` below);
  - workstation: abinit-fallbacks `b6aa2ef` (`my_develop`), and EasyBuild modules.

  Packages, versions and build: see [abinit-fallbacks on Lucia (six toolchains)](/post/abinit-fallbacks-libraries-lucia.html).
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

## 3.1 Cray Programming Environment

Commit `41f1b841a2` (rebased: `7087cbaa14`) lets configure find everything a Cray PE provides:

- **Compilers** (`config/m4/arch-mpi.m4`). By default, configure searches `PATH` for MPI wrappers (`config/m4/sd_arch_mpi.m4`), and in every PrgEnv it finds cray-mpich's own `mpicc`/`mpic++`/`mpifort`. Those neither link LibSci nor add the `craype-x86-milan` target options. When `CRAYPE_VERSION` and `PE_ENV` are set and `cc`/`CC`/`ftn` exist, configure now uses the craype wrappers unless `CC`/`CXX`/`FC` are given:

  ```
  checking for a Cray Programming Environment... yes (PrgEnv: GNU)
  configure: using the Cray compiler wrappers: CC=cc CXX=CC FC=ftn
  ```

- **`libsci` linear-algebra flavor** (`sd_math_linalg*.m4`), tried first by `auto` on a Cray PE, or selected with `--with-linalg-flavor=libsci`. Without it, the automatic detection picks the system OpenBLAS (GNU), MKL (Intel, since PrgEnv-intel ships one) or nothing (CCE, AOCC).
  - With the wrappers, no library has to be named: BLAS, LAPACK, BLACS and ScaLAPACK are all "none required".
  - With other compilers (e.g. `FC=mpifort`), the `libsci_<PrgEnv>` and `libsci_<PrgEnv>_mpi` libraries are linked explicitly, but only when they match the compiler in use.
- **BigDFT link order** (`sd_bigdft.m4`). `libyaml.a` is now passed as `-Wl,<path>/libyaml.a`. The Cray `ftn` driver moves plain file arguments in front of all `-l` options. That put `libyaml.a` *before* `libbigdft-1.a`, so with CCE every BigDFT link failed with `undefined reference to yaml_parser_initialize`.

## 3.2 AOCC 4.1 (classic flang)

AOCC's `ftn --version` prints `AMD clang version 16.0.3 (CLANG: AOCC_4.1.0...)`. That matches ABINIT's LLVM-flang test, and AOCC flang is still treated as LLVM because it needs the LLVM settings. The same commit (`lang-fortran.m4`, `configure.ac`) adapts two of them:

- `-ffixed-line-length=132` becomes `-Mextend` in its Fortran hints. Classic flang rejects the former (`clang-16: error: unknown argument: '-ffixed-line-length=132'`), and configure stops with "cannot compile a simple Fortran program".
- The linear-algebra Fortran flags (the ELPA modules) are added to `FCFLAGS` for every directory. Classic flang needs module files that are only used *indirectly* (`elpa_api.mod`), and ABINIT otherwise passes them only to the directories that use linear algebra directly. Commit `f7fb136e1c` extends this to LLVM Flang (section 3.4).

Commit `0d793ca5da` (rebased: `8e703f5ffa`) rewrites three places of ABINIT's own sources that classic flang cannot compile:

| File | Problem | Fix |
|---|---|---|
| `shared/libpaw/src/m_paw_atom_solve.F90` | `type(logical)` (Fortran 2008 `TYPE(intrinsic-type)`) is not supported: `F90-S-0155-Derived type has not been declared - logical` | `logical` |
| `src/62_ctqmc/m_CtqmcInterface.F90` | Bug in flang's integrated preprocessor: the nested macro `MALLOC` → `ABI_MALLOC` comes out as `C(this%Hybrid_chains,(this%num_chains))` | `ABI_MALLOC` directly |
| `src/48_diago/m_chebfi2.F90` (3 places) | Same preprocessor bug inside `ABI_MALLOC_IFNOT` | `if (.not. allocated(x)) then; ABI_MALLOC(x,...); endif` |

The preprocessor bug depends on the data: the same macros expand correctly in hundreds of other places, and in small test files.

## 3.3 `PSolver` argument

Commit `892bc37de3` (rebased: `15456e01e3`) fixes a real bug that only CCE detected. `m_mklocl_realspace.F90` called BigDFT's `PSolver` with the integer `0` where the interface expects a `type(xc_info)` (`ftn-1279`). The other compilers accepted it, so `PSolver` read an integer as an `xc_info` object. It now passes an object initialised with `xc_init(xc, 0, XC_ABINIT, 1)`.

## 3.4 OpenBLAS + ScaLAPACK, LLVM Flang and CMake HDF5

Five commits, each found with a failing build on the workstation (section 6):

- **LAPACK in OpenBLAS** (`7f3fa29eb3`, `sd_math_linalg_core.m4`). With OpenBLAS, LAPACK was always looked for as `-llapack`. OpenBLAS usually includes LAPACK, and EasyBuild installs no `liblapack`, so the serial detection failed; where a system `liblapack` exists, that reference LAPACK would be linked instead. OpenBLAS's own LAPACK is now used when present, and `-llapack` only when it is missing. This is limited to the `openblas` flavors, because IBM ESSL has routines with LAPACK names but other arguments.
- **BLACS in ScaLAPACK** (same commit). The `netlib` flavor looked for BLACS only as `-lblacs -lblacsCinit -lblacsF77init`, the layout of ScaLAPACK 1.x. ScaLAPACK ≥ 2.0 includes BLACS. When these libraries do not work, BLACS is now looked for in the ScaLAPACK libraries. Before, they also stayed in `LIBS` after the failed check, so the ScaLAPACK check failed as well. With these two changes, `eb_foss` should need no argument any more (section 4).
- **LLVM optimization flags** (`77acf5425e`, `config/optim/*_llvm_default-optim.conf`). The standard flags were `-mtune=native -mcpu=native`, which clang and flang only accept on AArch64. On x86_64, LLVM 20 stops with `unsupported option '-mcpu='`, and configure fails at its first link test ("linking to Fortran libraries from C fails"). The LLVM compilers now get `-march=native`, as GNU does, and AArch64 keeps the previous flags. AOCC uses these flags too, so `cray_aocc` now gets `-march=native` instead of `-mtune=native -mcpu=native`.
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

Beyond that:

| Toolchain | Extra arguments |
|---|---|
| `cray_cray`, `cray_gnu`, `cray_intel`, `cray_aocc` | **none** (FFTW3 included, see 5.3) |
| `eb_intel` | none |
| `eb_foss` | `--with-linalg-flavor=easybuild+elpa` at `892bc37de3`; none expected since `7f3fa29eb3` (section 3.4), not yet checked on Lucia |

**`--with-linalg-flavor=easybuild+elpa` (eb_foss, at `892bc37de3`).** The automatic detection finds EasyBuild's OpenBLAS (through pkg-config) but no ScaLAPACK. Its MPI flavor `netlib` hard-codes `-lblacs -lblacsCinit -lblacsF77init`, which ScaLAPACK 2.2.2 does not have. The `easybuild` flavor uses `-lopenblas -lscalapack`. `+elpa` is needed because an explicit flavor no longer adds the ELPA check by itself (`sd_math_linalg.m4`). The same OpenBLAS + ScaLAPACK stack is found without argument since `7f3fa29eb3`: see `foss` in section 6.

ABINIT's configure takes only 20–60 s here, so each configure attempt was a short Slurm job, and its logs were archived before the next attempt.

# 5. Results on Lucia

## 5.1 What "OK" means

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

## 5.2 Results

All six builds pass every check (`RESULT: OK`). They used `make -j 64` on one debug node:

| Toolchain | Linear algebra (configure / `ldd`) | FFT | `make` |
|---|---|---|---|
| `cray_cray` | `elpa+libsci` / `libsci_cray(_mpi).so.6` | FFTW3 (cray-fftw 3.3.10.8, + FFTW3-MPI) | 32.5 min |
| `cray_gnu` | `elpa+libsci` / `libsci_gnu(_mpi).so.6` | FFTW3 (cray-fftw 3.3.10.8, + FFTW3-MPI) | 8.3 min |
| `cray_intel` | `elpa+libsci` / `libsci_intel(_mpi).so.6` (no MKL) | FFTW3 (cray-fftw 3.3.10.8, + FFTW3-MPI) | 20.2 min |
| `cray_aocc` | `elpa+libsci` / `libsci_aocc(_mpi).so.6` | FFTW3 (cray-fftw 3.3.10.8, + FFTW3-MPI) | 26.2 min |
| `eb_intel` | `elpa+mkl` / MKL 2025.1 (`libmkl_scalapack_lp64`, `libmkl_blacs_intelmpi_lp64`, …) | DFTI | 9.5 min |
| `eb_foss` | `easybuild+elpa` / OpenBLAS 0.3.29, ScaLAPACK 2.2.2, FlexiBLAS 3.4.5 | FFTW3 (EasyBuild FFTW 3.3.10, found with pkg-config) | 9.1 min |

All times are clean builds. The four Cray builds are clean builds of commit `892bc37de3`, with `cray-fftw`. For the two EasyBuild toolchains, the `PSolver` fix (one file) came after their clean build, so they were brought up to date with an incremental `make`, after which `check.sh` passes again.

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

## 6.1 Toolchains and scenarios

| Name | Module | Compilers | MPI | Math library |
|---|---|---|---|---|
| `foss` | `foss/2025b` | GCC 14.3 | OpenMPI 5.0.8 | FlexiBLAS 3.4.5 / OpenBLAS 0.3.30, ScaLAPACK 2.2.2, FFTW 3.3.10 |
| `intel` | `intel/2025b` | icx/ifx 2025.2 | Intel MPI 2021.16 | MKL 2025.2 |
| `lfoss` | `lfoss/2025b` | LLVM 20.1.8 (clang + flang) | OpenMPI 5.0.8 | FlexiBLAS 3.4.5 / OpenBLAS 0.3.30, ScaLAPACK 2.2.2, FFTW 3.3.10 |

For each toolchain, EasyBuild also provides HDF5 1.14.6, netCDF 4.9.3 and netCDF-Fortran 4.6.2 (all with parallel I/O), LibXC 7.0.0 and ELPA 2025.06.002, but not Wannier90, XMLF90, libPSML or BigDFT. The scenarios differ in the modules loaded besides the toolchain:

| Scenario | Modules loaded besides the toolchain | Built by the fallbacks |
|---|---|---|
| `fb` | none | all the libraries |
| `mix` | `HDF5/1.14.6 netCDF/4.9.3 netCDF-Fortran/4.6.2 libxc/7.0.0 ELPA/2025.06.002` | Wannier90, XMLF90, libPSML, BigDFT |
| `partial` (`foss` only) | `HDF5/1.14.6 libxc/7.0.0` | netCDF, netCDF-Fortran, ELPA, Wannier90, XMLF90, libPSML, BigDFT |

The fallbacks themselves decide which libraries to build, from the modules loaded (see the fallbacks report). ABINIT then gets `--with-<package>=$FB/<package>/default` for each library installed by the fallbacks, and **no other argument**. For example, `foss` mix:

```bash
env -i HOME=$HOME USER=$USER LOGNAME=$LOGNAME TERM=$TERM PATH=/usr/bin:/bin \
    bash --noprofile --norc
source /etc/profile
module load foss/2025b
module load HDF5/1.14.6 netCDF/4.9.3 netCDF-Fortran/4.6.2 libxc/7.0.0 ELPA/2025.06.002

(cd $FB && <abinit-fallbacks>/configure --prefix=$FB && make && make install)

../configure \
  --with-wannier90=$FB/wannier90/default --with-xmlf90=$FB/xmlf90/default \
  --with-libpsml=$FB/libpsml/default --with-bigdft=$FB/bigdft/default
make -j 8
```

Smoke tests, with `tests/runtests.py`:

- serial: `v1[1]` and `libxc[1]`;
- 4 MPI processes: `paral[24]` (`paral_kgb 1` with `use_slk 1`, i.e. BLACS and ScaLAPACK) and `mpiio[22]` (MPI-IO).

## 6.2 Results

All seven builds succeed and pass the four smoke tests, all with commit `6cd90d43b2`:

| Build | Other arguments | Linear algebra (configure) | FFT | Found in the environment | `make -j 8` | Smoke tests |
|---|---|---|---|---|---|---|
| `foss` fb | none | `elpa+netlib+openblas` | FFTW3 | — | 5 min | 4/4 |
| `foss` mix | none | `elpa+netlib+openblas` | FFTW3 | HDF5, netCDF, netCDF-Fortran, LibXC, ELPA | 5 min | 4/4 |
| `foss` partial | none | `elpa+netlib+openblas` | FFTW3 | HDF5, LibXC | 5 min | 4/4 |
| `intel` fb | none | `elpa+mkl` | DFTI | — | 6 min | 4/4 |
| `intel` mix | none | `elpa+mkl` | DFTI | HDF5, netCDF, netCDF-Fortran, LibXC, ELPA | 6 min | 4/4 |
| `lfoss` fb | none | `elpa+netlib+openblas` | FFTW3 | — | 11 min | 4/4 |
| `lfoss` mix | none | `elpa+netlib+openblas` | FFTW3 | HDF5, netCDF, netCDF-Fortran, LibXC, ELPA | 11 min | 4/4 |

"Other arguments" means arguments besides the `--with-<package>` paths of the fallbacks: nine in `foss` fb and `intel` fb, eight in `lfoss` fb (no BigDFT), seven in `foss` partial, four in the `mix` builds (three with `lfoss`). FFTW3 and MKL come with the toolchain modules.

Every build:

- `abinit -b` lists `HAVE_MPI_IO HAVE_HDF5_MPI HAVE_NETCDF_MPI HAVE_NETCDF_FORTRAN_MPI HAVE_LIBXC HAVE_LINALG_SCALAPACK HAVE_LINALG_ELPA HAVE_WANNIER90 HAVE_LIBPSML HAVE_XMLF90`, plus `HAVE_BIGDFT` except with LLVM Flang;
- `ldd abinit` shows no linear algebra from the system, only from EasyBuild;
- the libraries of the fallbacks are static, so `ldd abinit` only lists libraries of EasyBuild and of the system.

## 6.3 Observations

- **No `--with-linalg-flavor`.** With GNU and LLVM, configure selects `elpa+netlib+openblas`: OpenBLAS with its own LAPACK, and BLACS and ScaLAPACK from `-lscalapack` (`checking netlib ScaLAPACK libraries for BLACS... -lscalapack`). With Intel, it selects `elpa+mkl`.
- **Libraries of EasyBuild are found without argument.** With the modules loaded, configure takes HDF5, netCDF, netCDF-Fortran, LibXC and ELPA from pkg-config (`checking how HDF5 parameters have been set... pkg`), with parallel I/O for HDF5 and netCDF.
- **OpenBLAS and FlexiBLAS.** ABINIT links OpenBLAS directly, while the fallbacks select FlexiBLAS. EasyBuild's ScaLAPACK is linked against FlexiBLAS, so both libraries end up in `ldd abinit`.
  - In the `fb` and `partial` builds, the linear algebra libraries are `-lelpa -lscalapack -lopenblas`: `libopenblas` is a direct dependency of `abinit`, and FlexiBLAS is only loaded by ScaLAPACK, so ABINIT's BLAS calls go to OpenBLAS.
  - In the `mix` builds, the pkg-config flags of EasyBuild's ELPA (`-lelpa -lscalapack -lflexiblas`) come before `-lopenblas`, so the BLAS calls go through FlexiBLAS, whose default backend is the same OpenBLAS.
- **LLVM Flang 20** builds ABINIT with the commits `77acf5425e`, `a84e3a012d` and `f7fb136e1c`, without BigDFT, which the fallbacks do not build with LLVM Flang. Its build takes about twice as long as with GCC.
- **One HDF5 for the whole I/O stack.** In `foss` partial, the fallbacks build netCDF against EasyBuild's HDF5, and ABINIT links that static netCDF with EasyBuild's shared HDF5 (commit `6cd90d43b2`).

# 7. Pitfalls worth knowing

- **Automake rebuild rules.** ABINIT's Makefiles have no "maintainer mode". If a file of `config/m4` is newer than `configure`, every `make` reruns aclocal/autoconf/automake *in the shared source tree*, and concurrent builds collide (`autom4te: cannot rename …`).
  - `aclocal` does not rewrite `aclocal.m4` when only the content of an included m4 file changes. So after `./autogen.sh`, touch `aclocal.m4`, `config.h.in`, `configure` and the `Makefile.in` files, in that order.
- **MPI launch inside a Slurm job** (Slurm 25.11, no default MPI plugin). Each library needs its own plugin, or the ranks start as separate singletons:
  - Cray MPICH: `srun --mpi=cray_shasta`;
  - Intel MPI: `srun --mpi=pmi2` with `I_MPI_PMI_LIBRARY=/usr/lib64/libpmi2.so`;
  - OpenMPI 5: `srun --mpi=pmix`.

# 8. Reproducing one build on Lucia

For example `cray_gnu`, from a build directory inside the ABINIT source tree on branch `cray_autodetect`:

```bash
env -i HOME=$HOME USER=$USER LOGNAME=$LOGNAME TERM=$TERM PATH=/usr/bin:/bin \
    bash --noprofile --norc
source /etc/profile
module --force purge
module load Cray/24.07 PrgEnv-gnu/8.4.0 cray-fftw/3.3.10.8
FB=~/program/abinit-fallbacks/_build_cray_gnu      # the fallbacks of this toolchain

../configure \
  --with-hdf5=$FB/hdf5/default --with-netcdf=$FB/netcdf4/default \
  --with-netcdf-fortran=$FB/netcdf4_fortran/default --with-libxc=$FB/libxc/default \
  --with-elpa=$FB/elpa/default --with-wannier90=$FB/wannier90/default \
  --with-xmlf90=$FB/xmlf90/default --with-libpsml=$FB/libpsml/default \
  --with-bigdft=$FB/bigdft/default
make -j 64
src/98_main/abinit -b      # check the CPP options
```

# 9. Open points

- **Lucia with the current branch.** Rebuild the six toolchains from the rebased branch with the commits of section 3.4. `eb_foss` should need no argument, and `cray_aocc` now gets `-march=native` instead of `-mtune=native -mcpu=native`.
- **No linear algebra is only a warning.** When nothing works, configure prints `linear algebra does not work` and goes on (policy `warn`), so the error only appears when the executables are linked, after the whole build.
- **`FCFLAGS_HINTS`** cannot override the vendor hints. `configure.ac` applies the user's value (`ABI_ENV_RECALL`, line 136) *before* computing the vendor hints (`ABI_FC_HINTS`, line 403), which overwrite it. With the AOCC changes of section 3.2, Lucia no longer needs it.
- **Upstream report** of AOCC flang's preprocessor bug.
- **FFTW variant at run time on Cray.** By default the loader picks the `x86_rome` build of `cray-fftw` (section 5.3). It would be cleaner to embed the module's library path in the binary, for example by linking with `CRAY_ADD_RPATH=yes`. That has not been tried yet.
- **FFTW3 threads.** OpenMP is off in these builds, so ABINIT uses `fftw3`, not `fftw3-threads`, although `cray-fftw` provides the threaded libraries.
