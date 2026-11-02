---
title: abinit-fallbacks on Lucia (six toolchains)
date: '2026-10-08 15:00:00'
permalink: /post/abinit-fallbacks-libraries-lucia.html
layout: post
published: true
---



# abinit-fallbacks on Lucia (six toolchains)

> **Status, 2026-11-02.** abinit-fallbacks builds all the external libraries of ABINIT from source, as static libraries, on the six toolchains of Lucia. The current `my_develop` was also tested on a workstation with three EasyBuild 2025b toolchains: with no configure argument, it uses the libraries EasyBuild provides and builds only the missing ones (section 8). ABINIT itself, built against these libraries, is described in [ABINIT on Lucia (six toolchains)](/post/abinit-fallbacks-lucia.html).

Versions:

- abinit-fallbacks, fork [tsaihsiaoyi/abinit-fallbacks](https://github.com/tsaihsiaoyi/abinit-fallbacks):
  - Lucia (sections 1–7): built and checked at commit `6266085` (branch `bigdft_xc_fix`, three commits on top of `my_develop` at `3ddec17`), now merged into `my_develop`;
  - `my_develop` has moved on since. In particular, commit `2a4a80a` switches to BigDFT abinit-1.7.1.34, which fixes problems 1 and 2 of section 5 upstream, and commits `ee852b4`…`b6aa2ef` use the packages already available in the build environment. These later commits have not been built on Lucia;
  - workstation (section 8): `my_develop` at `b6aa2ef`.
- Machines:
  - Lucia (Cenaero), debug nodes with 2 × AMD EPYC 7763 (Zen 3, AVX2, no AVX-512), RHEL 8, Slurm 25.11;
  - a workstation with an Intel Core Ultra 7 265 (20 cores), 30 GB of memory, Ubuntu 24.04.5.

# 1. Goal and rules

For each of the six compiler toolchains available on Lucia, build all the ABINIT external libraries from source with abinit-fallbacks, so that ABINIT can then be built against exactly those libraries.

- Only the **compiler, MPI and math library** come from cluster modules. HDF5, netCDF, LibXC, ELPA, Wannier90, BigDFT, … are all built here, never taken from `cray-hdf5`, `cray-netcdf` or EasyBuild modules. None of the fallbacks uses FFTW.
- Everything runs in a **clean environment**: `env -i`, then `source /etc/profile` (only for the `module` command), then the module lines of the toolchain. My `~/.bashrc` activates a conda environment, and two hidden dependencies came from it before I switched to `env -i`:
  - netCDF-C picked up conda's `xml2-config`;
  - BigDFT needed conda's `python`.
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

# 3. Packages

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
| BigDFT | abinit-1.7.1.33 |
| AtomPAW | 4.2.0.3 (program only) |

# 4. Changes to the abinit-fallbacks build system

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

# 5. BigDFT: three more fixes, needed to link it into ABINIT

These are in branch `bigdft_xc_fix`. Each one surfaced only once ABINIT was configured with `--with-bigdft`:

1. **Private routines used from outside** (patch `bigdft-abinit-1.7.1.33-0001.patch`, commit `46c9322`). BigDFT's `src/modules/xc.f90` calls `abi_libxc_functionals_getrefs` and `abi_libxc_functionals_constants_load`, but the bundled libABINIT declares them `private`. The compilers then treat the calls as calls to external procedures that don't exist, so **every** program linking `libbigdft-1.a` fails:

   ```
   undefined reference to `abi_libxc_functionals_constants_load_'
   ```

   The patch makes the two routines public.

2. **Missing module files** (patch `-0002`, commit `f36663b`). BigDFT installs its own `.mod` files but not those of the bundled libABINIT (`abi_libxc_functionals.mod`, …). Classic flang (AOCC) needs every module used indirectly, so the patch adds an install rule for them.

3. **Fake ScaLAPACK** (commit `6266085`). BigDFT's configure only tries `-lscalapack`. Without it, BigDFT compiles `src/modules/blacs_fake.f90`, stubs such as `blacs_gridinit` or `pdgemm` that just `stop 'FAKE …'`. In a program linking `libbigdft-1.a`, these stubs **replace the real routines** whenever the real library comes later on the link line. That is the case with the Cray wrappers, which add LibSci at the very end. Every ScaLAPACK run of ABINIT then stopped in `FAKE BLABS_GRIDINIT`. The fallbacks now pass `--with-scalapack=-lsci_<PrgEnv>_mpi` (LibSci) or `-lmkl_scalapack_lp64` (MKL) to BigDFT.

BigDFT abinit-1.7.1.34 includes fixes 1 and 2, so `my_develop` no longer carries the two patches since commit `2a4a80a`. Fix 3 stays on the fallbacks side.

# 6. Build

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

# 7. Results

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

# 8. Workstation tests: libraries from EasyBuild and from the fallbacks

## 8.1 Setup

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

## 8.2 What configure selects

- **Compilers:** the MPI wrappers, `mpicc`/`mpicxx`/`mpif90` for `foss` (GNU 14.3) and `lfoss` (LLVM 20.1), `mpiicx`/`mpiicpx`/`mpiifx` for `intel`.
- **Linear algebra:** `flexiblas+scalapack_pkg` (`-lscalapack -lflexiblas`) for `foss` and `lfoss`, `mkl` (`-qmkl=cluster`) for `intel`. The internal LINALG fallback is never built.
- **External packages:** each one is looked for with pkg-config (`hdf5_hl`, `libxc`, `elpa`, `netcdf`, `netcdf-fortran`) and used instead of the fallback when it works with the current compilers. The summary of configure marks it with "Install: no".

| Build | Built by the fallbacks | Taken from EasyBuild | Time |
|---|---|---|---|
| `foss` fb | all ten packages | none | 4 min |
| `foss` mix | Wannier90, XMLF90, libPSML, BigDFT, AtomPAW | HDF5, netCDF, netCDF-Fortran, LibXC, ELPA | 1 min |
| `foss` partial | netCDF, netCDF-Fortran, ELPA, Wannier90, XMLF90, libPSML, BigDFT, AtomPAW | HDF5, LibXC | 2 min |
| `intel` fb | all ten packages | none | 6 min |
| `intel` mix | Wannier90, XMLF90, libPSML, BigDFT, AtomPAW | HDF5, netCDF, netCDF-Fortran, LibXC, ELPA | 3 min |
| `lfoss` fb | all but BigDFT | none | 5 min |
| `lfoss` mix | Wannier90, XMLF90, libPSML, AtomPAW | HDF5, netCDF, netCDF-Fortran, LibXC, ELPA | 1 min |

Notes:

- In `foss` partial, netCDF-C is built against the HDF5 of EasyBuild, so the whole I/O stack uses one HDF5: its `libnetcdf.settings` lists `-L<EasyBuild HDF5>/lib -lhdf5_hl -lhdf5`, with NC-4 parallel support.
- BigDFT is not built with LLVM Flang, which crashes on its source code (`LLVM Flang cannot compile BigDFT => disabling the BigDFT fallback`).
- ABINIT built against each of these seven installations, and passed its smoke tests: see section 6 of the [ABINIT report](/post/abinit-fallbacks-lucia.html).

# 9. Open points

- **Rebuild from the current `my_develop`** (BigDFT abinit-1.7.1.34 and the later build-system changes) on the six toolchains of Lucia.
- **Fake ScaLAPACK in BigDFT** (fix 3 of section 5). BigDFT's configure could look for ScaLAPACK under other names, so that its stubs are never built when a real library is available.
- **C compiler type of `mpiicx`.** configure reports `checking for compiler type of 'mpiicx'... gnu`, with no version. The oneAPI check only reads the first line of `mpiicx -V`, which is the banner of the Intel MPI wrapper (`mpiicx for the Intel(R) MPI Library 2021.16`), and the GNU check matches afterwards. The tricks use the Fortran compiler (`intel 2025.2`) and MKL is selected, so this had no effect on the builds of section 8.
