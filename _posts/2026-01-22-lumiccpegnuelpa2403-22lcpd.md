---
title: LUMI-C-cpeGNU-elpa-24.03
date: '2026-01-22 12:57:29'
permalink: /post/lumiccpegnuelpa2403-22lcpd.html
layout: post
published: true
---



# LUMI-C-cpeGNU-elpa-24.03

> **This recipe is outdated due to** **[update](https://lumi-supercomputer.github.io/LUMI-training-materials/User-Updates/Update-202601/)** **of the cluster during at January 21 2026.**

# 1. patch for LUMI compilation

First need a patch for being compatible with the cray-gnu compiler.

```patch
diff --git a/src/61_occeig/m_fstab.F90 b/src/61_occeig/m_fstab.F90
index 85cf9ee5ac..be45b534bf 100644
--- a/src/61_occeig/m_fstab.F90
+++ b/src/61_occeig/m_fstab.F90
@@ -191,7 +191,7 @@ contains  !============================================================
 subroutine fstab_free(fstab)
 
 !Arguments ------------------------------------
- class(fstab_t),intent(inout) :: fstab
+ class(fstab_t),intent(inout),target :: fstab
 ! ************************************************************************
 
  ! integer
@@ -253,7 +253,7 @@ subroutine fstab_init(fstab, ebands, cryst, dtset, tetra, comm)
  type(htetra_t),intent(out) :: tetra
  integer,intent(in) :: comm
 !arrays
- type(fstab_t),target,intent(out) :: fstab(ebands%nsppol)
+ class(fstab_t),target,intent(out) :: fstab(ebands%nsppol)
 
 !Local variables-------------------------------
 !scalars
@@ -265,7 +265,7 @@ subroutine fstab_init(fstab, ebands, cryst, dtset, tetra, comm)
  logical :: in_win
  character(len=80) :: errstr
  character(len=5000) :: msg
- type(fstab_t),pointer :: fs
+ class(fstab_t),pointer :: fs
  type(krank_t) :: krank
 !arrays
  integer :: kptrlatt(3,3)
@@ -534,7 +534,7 @@ integer function fstab_findkg0(fstab, kpt, g0) result(ik_fs)
 
 !Arguments ------------------------------------
 !scalars
- class(fstab_t),intent(in) :: fstab
+ class(fstab_t),intent(in),target :: fstab
 !arrays
  integer,intent(out) :: g0(3)
  real(dp),intent(in) :: kpt(3)
diff --git a/src/78_eph/m_phgamma.F90 b/src/78_eph/m_phgamma.F90
index 5b912ec1c7..22b18cb8eb 100644
--- a/src/78_eph/m_phgamma.F90
+++ b/src/78_eph/m_phgamma.F90
@@ -544,7 +544,7 @@ subroutine phgamma_init(gams, cryst, ifc, ebands, fstab, dtset, eph_scalprod, ng
  type(crystal_t),intent(in) :: cryst
  type(ifc_type),intent(in) :: ifc
  type(ebands_t),intent(in) :: ebands
- type(fstab_t), intent(in) :: fstab
+ class(fstab_t), intent(in) :: fstab
  type(dataset_type),intent(in) :: dtset
 !arrays
  integer,intent(in) :: ngqpt(3)
@@ -3040,7 +3040,7 @@ subroutine eph_phgamma(wfk0_path, dtfil, ngfft, ngfftf, dtset, cryst, ebands, dv
  real(dp) :: edos_step, edos_broad, sigma, ecut, eshift, eig0nk
  logical :: gen_eigenpb, need_velocities, isirr_k, isirr_kq, print_time_k, need_ftinterp
  type(wfd_t) :: wfd
- type(fstab_t),pointer :: fs
+ class(fstab_t),pointer :: fs
  type(gs_hamiltonian_type) :: gs_hamkq
  type(rf_hamiltonian_type) :: rf_hamkq
  type(edos_t) :: edos
@@ -3071,7 +3071,7 @@ subroutine eph_phgamma(wfk0_path, dtfil, ngfft, ngfftf, dtset, cryst, ebands, dv
  real(dp),allocatable :: wt_ek(:,:), wt_ekq(:,:), dbldelta_wts(:,:)
  real(dp),allocatable :: tgamvv_in(:,:,:,:),  vv_kk(:,:,:), tgamvv_out(:,:,:,:), vv_kkq(:,:,:), tmp_vals_ee(:,:,:,:,:), emesh(:)
  logical,allocatable :: bks_mask(:,:,:),keep_ur(:,:,:)
- type(fstab_t),target,allocatable :: fstab(:)
+ class(fstab_t),target,allocatable :: fstab(:)
  type(pawcprj_type),allocatable  :: cwaveprj0(:,:)
 #ifdef HAVE_MPI
  integer :: ndims, comm_cart, me_cart, coords(5)
@@ -3118,7 +3118,7 @@ subroutine eph_phgamma(wfk0_path, dtfil, ngfft, ngfftf, dtset, cryst, ebands, dv
 
  ! Find Fermi surface k-points
  ! TODO: support kptopt, change setup of k-points if tetra: fist tetra weights then k-points on the Fermi surface!
- ABI_MALLOC(fstab, (nsppol))
+ allocate(fstab(nsppol))
  call fstab_init(fstab, ebands, cryst, dtset, tetra, comm)
  call tetra%free()
 
@@ -4077,9 +4077,10 @@ subroutine eph_phgamma(wfk0_path, dtfil, ngfft, ngfftf, dtset, cryst, ebands, dv
  ABI_SFREE(tgamvv_out)
  call ddkop%free(); call gs_hamkq%free(); call wfd%free()
  do spin=1,ebands%nsppol
-   call fstab(spin)%free()
+   fs => fstab(spin)
+   call fs%free()
  end do
- ABI_FREE(fstab)
+ deallocate(fstab)
  call pawcprj_free(cwaveprj0)
  ABI_FREE(cwaveprj0)
 
@@ -4185,7 +4186,7 @@ subroutine phgamma_setup_qpoint(gams, fs, cryst, ebands, spin, ltetra, qpt, nest
 
 !Arguments ------------------------------------
  class(phgamma_t),intent(inout) :: gams
- type(fstab_t),intent(inout) :: fs
+ class(fstab_t),intent(inout),target :: fs
  type(crystal_t),intent(in) :: cryst
  type(ebands_t),intent(in) :: ebands
  integer,intent(in) :: spin, ltetra, comm
diff --git a/src/78_eph/m_wkk.F90 b/src/78_eph/m_wkk.F90
index 8eb4bff660..096c52d0e3 100644
--- a/src/78_eph/m_wkk.F90
+++ b/src/78_eph/m_wkk.F90
@@ -161,7 +161,8 @@ subroutine wkk_run(wfk0_path, dtfil, ngfft, ngfftf, dtset, cryst, ebands, wfk_hd
  complex(gwp),allocatable :: vc_sqrt_gx(:), ur_nkp(:,:), ur_mk(:,:), kxcg(:,:), mu_mn(:,:)
  complex(dp),allocatable :: w_ee(:,:) ! w_kkp(:,:,:,:)
  logical,allocatable :: bks_mask(:,:,:), keep_ur(:,:,:)
- type(fstab_t),target,allocatable :: fstab(:)
+ class(fstab_t),target,allocatable :: fstab(:)
+ class(fstab_t), pointer :: fs
 !************************************************************************
 
  ABI_CHECK(psps%usepaw == 0, "PAW not implemented")
@@ -191,7 +192,7 @@ subroutine wkk_run(wfk0_path, dtfil, ngfft, ngfftf, dtset, cryst, ebands, wfk_hd
 
  ! Find Fermi surface k-points.
  ! TODO: support kptopt, change setup of k-points if tetra: fist tetra weights then k-points on the Fermi surface!
- ABI_MALLOC(fstab, (nsppol))
+ allocate(fstab(nsppol))
  call fstab_init(fstab, ebands, cryst, dtset, tetra, comm)
  call tetra%free()
 
@@ -681,9 +682,10 @@ subroutine wkk_run(wfk0_path, dtfil, ngfft, ngfftf, dtset, cryst, ebands, wfk_hd
  call epsm1%free()
 
  do spin=1,ebands%nsppol
-   call fstab(spin)%free()
+   fs => fstab(spin)
+   call fs%free()
  end do
- ABI_FREE(fstab)
+ deallocate(fstab)
 
  call xmpi_barrier(comm) ! This to make sure that the parallel output of WKK is completed
  call cwtime_report(" wkk_run: MPI barrier before returning.", cpu_all, wall_all, gflops_all, end_str=ch10, comm=comm)

```

# 2. easybuild dependencies

For scalapack can be supported by `cray-libsci`​. So we need to compile `ELPA`.

Use `eb`​ from easybuild to compile `libxc`​ and `ELPA`. Config file need to be changed.

For libxc:

```python
# contributed by Luca Marsella (CSCS)
# adapted for LUMI by Peter Larsson

local_Perl_version = '5.38.0'

easyblock = 'CMakeMake'

name = 'libxc'
version =  '7.0.0'

homepage = 'https://www.tddft.org/programs/libxc/'

whatis = [
    "Description: Libxc is a library of exchange-correlation and kinetic energy functionals " +
    "for density-functional theory."
]

description = """
Libxc is a library of exchange-correlation and kinetic energy functionals for
density-functional theory. The original aim was to provide a portable, well
tested and reliable set of LDA, GGA, and meta-GGA functionals.

Libxc is written in C, but it also comes with Fortran binding.

It is released under the MPL license (v. 2.0). In all publications resulting
from your use of Libxc, please cite:

[ref] Susi Lehtola, Conrad Steigemann, Micael J. T. Oliveira, and Miguel A. L. Marques,
      "Recent developments in Libxc - A comprehensive  library of functionals for
      density functional theory", Software X 7, 1 (2018)
"""

docurls = [
    'Manual: https://www.tddft.org/programs/libxc/manual/',
    'Available functionals: https://www.tddft.org/programs/libxc/functionals/'
]

toolchain = {'name': 'cpeGNU', 'version': '24.03'}
toolchainopts = {'opt': True}

source_urls = ['https://gitlab.com/%(name)s/%(name)s/-/archive/%(version)s']
sources = [SOURCE_TAR_BZ2]
#checksums = ['f72ed08af7b9dff5f57482c5f97bff22c7dc49da9564bc93871997cbda6dacf3']
#checksums = ['e9ae69f8966d8de6b7585abd9fab588794ada1fab8f689337959a35abbf9527d']  # libxc-7.0.0.tar.bz2

builddependencies = [
    ('buildtools', '%(toolchain_version)s', '', True), # For CMake
    ('craype-network-none', EXTERNAL_MODULE),
    ('craype-accel-host',   EXTERNAL_MODULE),
    ('Perl',                local_Perl_version),
]

local_common_configopts = '-DENABLE_FORTRAN=ON -DDISABLE_KXC=OFF' # --enable-kxc

configopts = [
    local_common_configopts + ' -DBUILD_SHARED_LIBS=OFF',
    local_common_configopts + ' -DBUILD_SHARED_LIBS=ON',
]

preconfigopts = 'module unload rocm cray-libsci && '
prebuildopts = preconfigopts

postinstallcmds = [
    'mkdir -p %(installdir)s/share/licenses/%(name)s',
    'cd ../%(name)s-%(version)s && cp AUTHORS CITATION COPYING NEWS README.md %(installdir)s/share/licenses/%(name)s',
]

sanity_check_paths = {
    'files': ['bin/xc-info'] +
             ['lib/libxc%s.%s' % (x, y) for x in ['', 'f03'] for y in ['a', SHLIB_EXT]],
    'dirs': ['include', 'lib/pkgconfig', 'lib/cmake/Libxc'],
}

moduleclass = 'chem'
```

For ELPA:

```python
# Contributed by Luca Marsella (CSCS)
# Some adaptations by Kurt Lust (University of Antwerpen and LUMI consortium)
easyblock = 'ConfigureMake'

local_ELPA_version =           '2024.05.001'     # Synchronized with CSCS but also the version in EB 2021b

name = 'ELPA'
version = local_ELPA_version
versionsuffix = '-CPU'

homepage = 'http://elpa.mpcdf.mpg.de'

whatis = [
    "Description: ELPA - Eigenvalue SoLvers for Petaflop-Applications",
]

description = """
The publicly available ELPA library provides highly efficient and highly
scalable direct eigensolvers for symmetric matrices. Though especially designed
for use for PetaFlop/s applications solving large problem sizes on massively
parallel supercomputers, ELPA eigensolvers have proven to be also very efficient
for smaller matrices. All major architectures are supported.

ELPA provides static and shared libraries with and without OpenMP support
and with and without MPI. GPU kernels are not included in this module.
"""

docurls = [
    'Manual pages in section 1 and 3'
]

toolchain = {'name': 'cpeGNU', 'version': '24.03'}
toolchainopts = {'usempi': True}

source_urls = ['https://elpa.mpcdf.mpg.de/software/tarball-archive/Releases/%(version)s/']
sources =  ['elpa-%(version)s.tar.gz']
checksums = ['9caf41a3e600e2f6f4ce1931bd54185179dade9c171556d0c9b41bbc6940f2f6']

builddependencies = [ # We use the system Python and Perl.
    ('buildtools',        '%(toolchain_version)s', '', True), # For Autotools and others.
    ('craype-accel-host', EXTERNAL_MODULE)
]

preconfigopts = 'module unload rocm && '
prebuildopts = preconfigopts

preconfigopts += 'CC=cc CXX=CC FC=ftn CPP=cpp && '
prebuildopts += 'make clean && '

local_common_configopts = ' '.join([
    'FCFLAGS="$FCFLAGS" CPP=cpp',
    '--disable-generic',
    '--disable-sse',
    '--disable-avx',
    '--enable-avx2',
    '--disable-avx512',
    '--enable-static',
    '--with-mpi=yes',
])

configopts = [
    local_common_configopts + ' --disable-openmp ',
    local_common_configopts + ' --enable-openmp '
]

sanity_check_paths = {
    'files': ['lib/libelpa.a', 'lib/libelpa.so', 
              'lib/libelpa_openmp.a', 'lib/libelpa_openmp.so'
              ],
    'dirs':  ['bin', 'lib/pkgconfig',
              'include/%(namelower)s-%(version)s/%(namelower)s', 'include/%(namelower)s-%(version)s/modules',
              'include/%(namelower)s_openmp-%(version)s/%(namelower)s', 'include/%(namelower)s_openmp-%(version)s/modules'
              ]
}

modextrapaths = {
    'CPATH': [
        'include/elpa_openmp-%(version)s', 
              'include/elpa-%(version)s']
}

modextravars = {
    'ELPA_INCLUDE_DIR': '%(installdir)s/include/elpa-%(version)s',
    'ELPA_OPENMP_INCLUDE_DIR': '%(installdir)s/include/elpa_openmp-%(version)s'
}

moduleclass = 'math'
```

Remember to `module purge`​ environment first. Then use `eb /path/to/config -r` to compile these.

After compilation of dependencies, the modulefiles should be auto generated at `~/EasyBuild/modules/LUMI/24.03/partition/C/`.

# 3. configure

```plaintext
┌─() [tsaihsia@uan03] - [/pfs/lustrep2/users/tsaihsia/program/abinit.worktrees/gwr/build] - [2025-12-19 13:50:17]
└─[130] $ module purge && module load LUMI/24.03 && module load partition/C && module load ELPA/2024.05.001-cpeGNU-24.03-CPU libxc/7.0.0-cpeGNU-24.03 cray-python/3.11.7 cray-fftw/3.3.10.7 cray-hdf5-parallel/1.12.2.11 cray-netcdf-hdf5parallel/4.9.0.11

┌─() [tsaihsia@uan03] - [/pfs/lustrep2/users/tsaihsia/program/abinit.worktrees/gwr/build] - [2025-12-19 13:50:25]
└─[0] $ ml

Currently Loaded Modules:
  1) ModuleLabel/label                    (S)  13) cray-mpich/8.1.29
  2) lumi-tools/24.05                     (S)  14) cray-libsci/24.03.0
  3) init-lumi/0.2                        (S)  15) PrgEnv-gnu/8.5.0
  4) LUMI/24.03                           (S)  16) gcc-native/13.2
  5) craype-x86-milan                          17) perftools-base/24.03.0
  6) craype-accel-host                         18) cpeGNU/24.03
  7) libfabric/1.15.2.0                        19) ELPA/2024.05.001-cpeGNU-24.03-CPU
  8) craype-network-ofi                        20) libxc/7.0.0-cpeGNU-24.03
  9) xpmem/2.8.2-1.0_5.1__g84a27a5.shasta      21) cray-python/3.11.7
 10) partition/C                          (S)  22) cray-fftw/3.3.10.7
 11) craype/2.7.31.11                          23) cray-hdf5-parallel/1.12.2.11
 12) cray-dsmml/0.3.0                          24) cray-netcdf-hdf5parallel/4.9.0.11
```

```bash
CC=cc CXX=CC FC=ftn FCLIBS=" " LINALG_LIBS="-lsci_gnu -lsci_gnu_mpi" ../configure --prefix=${workspaceFolder}/build --with-mpi=yes --enable-mpi-io=yes --with-linalg-flavor=netlib+elpa --with-fft-flavor=fftw3 --with-elpa=${EBROOTELPA} --with-libxc=${EBROOTLIBXC} --enable-memory-profiling --enable-mpi-inplace=yes --with-optim-flavor=standard --enable-gw-dpc=yes
```

Then comment the `HAVE_FFTW3_THREADS`​ in `config.h`.

This is because building system will automaticly define `HAVE_FFTW3_THREADS`​ when `-lfftw3_threads` working even flavor is fftw3 but not fftw3-threads.

In principle one should modify config:

```patch
diff --git a/config/m4/sd_math_fft.m4 b/config/m4/sd_math_fft.m4
index 598bdbfdac..28e76204a3 100644
--- a/config/m4/sd_math_fft.m4
+++ b/config/m4/sd_math_fft.m4
@@ -120,7 +120,7 @@ AC_DEFUN([SD_FFT_DETECT], [
             AC_DEFINE([HAVE_FFTW3_MPI], 1,
               [Define to 1 if you have a MPI-enabled FFTW3 library.])
           fi
-          if test "${sd_fftw3_threads_ok}" = "yes" ; then
+          if test "${sd_fft_flavor}" = "fftw3-threads" -a "${sd_fftw3_threads_ok}" = "yes" ; then
             AC_DEFINE([HAVE_FFTW3_THREADS], 1,
               [Define to 1 if you have a threads-enabled FFTW3 library.])
           fi
```

# 4. make

```bash
make -j 16 install
```

# 5. test

```bash
salloc -N 1 -c 128 --account=project_465002489 -t 00:30:00 -p debug
../tests/runtests.py -n 1 -j 128 --use-srun --mpi-args="--cpus-per-task=1" -t 900
```

```plaintext
Suite            failed  passed  succeeded  skipped  disabled  run_etime  tot_etime
atdep                 0       0         38        0         0    1025.40    1031.58
atompaw               0       0          0        2         0       0.00       0.01
bigdft                0       0          0       19         0       0.00       0.04
bigdft_paral          0       0          0        4         0       0.00       0.01
built-in              0       0          5        2         0      37.11      37.16
etsf_io               0       0          8        0         0      67.17      67.44
fast                  0       1         10        0         0     118.55     119.78
gpu                   0       0          0        8         0       0.00       0.04
gpu_kokkos            0       0          0        1         0       0.00       0.00
gpu_omp               0       0          0       50         0       0.00       0.12
gwpt_suite            0       0          0        1         0       0.00       0.01
gwr_suite             0       6          6        0         0     157.31     157.95
hpc_gpu_omp           0       0          0       13         0       0.00       0.02
libxc                 0       5         32        0         0     671.19     674.61
mpiio                 0       0          1       16         0       8.03       8.16
paral                 0       4         27      142         0     515.31     518.30
psml                  0       0          0       14         0       0.00       0.02
rttddft_suite         0       0          6        0         0      96.76      97.22
seq                   0       0          0       18         0       0.00       0.02
tutoatdep             0       0          5        0         0      26.87      27.23
tutomultibinit        0       0          6        4         0     343.51     347.81
tutoparal             0       0          4       24         0     157.56     157.84
tutoplugs             0       0          0        8         0       0.00       0.01
tutorespfn            0      11         18        4         0    1801.19    1805.65
tutorial              0      13         52        0         0    2324.27    2328.80
unitary               0       0         22       16         0      64.61      64.93
v1                    0       3         71        0         0     281.91     285.43
v10                   0       7         30        6         0    1280.76    1282.98
v2                    0      15         64        0         0     291.95     295.78
v3                    0      13         65        0         0     547.70     554.17
v4                    0      15         46        0         0     391.19     396.00
v5                    0      10         62        0         0     827.89     833.11
v6                    0      10         50        0         0     543.36     549.48
v67mbpt               0       3         25        0         0     336.32     340.07
v7                    0      18         47        0         0    1038.57    1046.34
v8                    1      13         52        5         0    1400.56    1408.54
v9                    0      18         88        2         0    1390.84    1401.72
vdwxc                 0       0          0        1         0       0.00       0.00
wannier90             0       0          0       10         0       0.00       0.01
```

All failed tests are due to known memory leakage.

‍
