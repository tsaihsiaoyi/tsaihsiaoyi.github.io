---
title: LUMI-C-cpeGNU-elpa
date: '2026-02-23 19:12:25'
permalink: /post/lumiccpegnuelpa-t1838.html
layout: post
published: true
---



# LUMI-C-cpeGNU-elpa

> This version of ABINIT (b71bbc9) is compiled to support scalapack + elpa + openmp at LUMI-C/25.04 .

# 1. patch for LUMI compilation

Good news: After [update](https://lumi-supercomputer.github.io/LUMI-training-materials/User-Updates/Update-202601/), seems we don't need patch anymore.

# 2. easybuild dependencies

For scalapack can be supported by `cray-libsci`​. So we need to compile `ELPA`.

Use `eb`​ from easybuild to compile `libxc`​ and `ELPA`. Config file need to be changed.

For libxc:

```python
# contributed by Luca Marsella (CSCS)
# adapted for LUMI by Peter Larsson

local_Perl_version = '5.40.0'

easyblock = 'CMakeMake'

local_libxc7_version =       '7.0.0'         # https://gitlab.com/libxc/libxc/-/releases

name =          'libxc'
version =       local_libxc7_version
versionsuffix = '-KXC'

homepage = 'https://libxc.gitlab.io/'

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
    'Manual: https://libxc.gitlab.io/manual/',
    'Available functionals: https://libxc.gitlab.io/functionals/'
]

toolchain = {'name': 'cpeGNU', 'version': '25.03'}
toolchainopts = {'opt': True}

source_urls = ['https://gitlab.com/%(name)s/%(name)s/-/archive/%(version)s']
sources = [SOURCE_TAR_BZ2]
#checksums = ['f72ed08af7b9dff5f57482c5f97bff22c7dc49da9564bc93871997cbda6dacf3']
checksums = ['e9ae69f8966d8de6b7585abd9fab588794ada1fab8f689337959a35abbf9527d']  # libxc-7.0.0.tar.bz2

builddependencies = [
    ('buildtools', '%(toolchain_version)s', '', True), # For CMake
    ('craype-network-none', EXTERNAL_MODULE),
    ('craype-accel-host',   EXTERNAL_MODULE),
    ('Perl',                local_Perl_version),
]

local_common_configopts = '-DENABLE_FORTRAN=ON -DCMAKE_INSTALL_LIBDIR=lib -DDISABLE_KXC=OFF'

configopts = [
    local_common_configopts + ' -DBUILD_SHARED_LIBS=OFF',
    local_common_configopts + ' -DBUILD_SHARED_LIBS=ON',
]

preconfigopts = 'module unload rocm cray-libsci && '
prebuildopts = pretestopts = preconfigopts

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

local_ELPA_version = '2025.06.001'

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

toolchain = {'name': 'cpeGNU', 'version': '25.03'}
toolchainopts = {'usempi': True}

source_urls = ['https://elpa.mpcdf.mpg.de/software/tarball-archive/Releases/%(version)s/']
sources =  ['elpa-%(version)s.tar.gz']
checksums = ['feeb1fea1ab4a8670b8d3240765ef0ada828062ef7ec9b735eecba2848515c94']

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

postinstallcmds = [
    'mkdir -p %(installdir)s/share/licenses/%(name)s',
    'cp -r Changelog CONTRIBUTING.md LICENSE README.md COPYING %(installdir)s/share/licenses/%(name)s',   
]

sanity_check_paths = {
    'files': ['lib/libelpa.a', 'lib/libelpa.so', 'lib/libelpa_openmp.a', 'lib/libelpa_openmp.so'],
    'dirs':  ['bin', 'lib/pkgconfig',
              'include/%(namelower)s-%(version)s/%(namelower)s', 'include/%(namelower)s-%(version)s/modules',
              'include/%(namelower)s_openmp-%(version)s/%(namelower)s', 'include/%(namelower)s_openmp-%(version)s/modules']
}

modextrapaths = {
    'CPATH': ['include/elpa_openmp-%(version)s', 'include/elpa-%(version)s']
}

modextravars = {
    'ELPA_INCLUDE_DIR': '%(installdir)s/include/elpa-%(version)s',
    'ELPA_OPENMP_INCLUDE_DIR': '%(installdir)s/include/elpa_openmp-%(version)s'
}

moduleclass = 'math'
```

Remember to `module purge`​ environment first. Then load easybuild environment with `module load LUMI/25.03 && module load partition/C && module load EasyBuild-user`​. Then use `eb /path/to/config -r` to compile these.

After compilation of dependencies, the modulefiles should be auto generated at `~/EasyBuild/modules/LUMI/25.03/partition/C/`.

# 3. configure

After compiling those dependencies, modules can be loaded as follow:

```bash
module purge && module load LUMI/25.03 && module load partition/C && module load ELPA/2025.06.001-cpeGNU-25.03-CPU libxc/7.0.0-cpeGNU-25.03-KXC cray-python/3.11.7 cray-fftw/3.3.10.10 cray-hdf5-parallel/1.14.3.5 cray-netcdf-hdf5parallel/4.9.0.17
```

You can check modules by running command `ml`​ or `module list`, results should be like:

```plaintext
Currently Loaded Modules:
  1) ModuleLabel/label              (S)  13) cray-dsmml/0.3.1
  2) lumi-tools/24.05               (S)  14) cray-mpich/8.1.32
  3) init-lumi/0.2                  (S)  15) cray-libsci/25.03.0
  4) LUMI/25.03                     (S)  16) PrgEnv-gnu/8.6.0
  5) craype-x86-milan                    17) perftools-base/25.03.0
  6) craype-accel-host                   18) cpeGNU/25.03
  7) libfabric/1.22.0                    19) ELPA/2025.06.001-cpeGNU-25.03-CPU
  8) craype-network-ofi                  20) libxc/7.0.0-cpeGNU-25.03-KXC
  9) xpmem/2.11.5-1.3_g73ade43320bc      21) cray-python/3.11.7
 10) partition/C                    (S)  22) cray-fftw/3.3.10.10
 11) gcc-native/14.2                     23) cray-hdf5-parallel/1.14.3.5
 12) craype/2.7.34                       24) cray-netcdf-hdf5parallel/4.9.0.17
```

Now let's configure. Like compiling any other software, make a new directory e.g. `mkdir build`, and run command below in this directory.

```bash
CC=cc CXX=CC FC=ftn FCLIBS=" " LINALG_LIBS="-lsci_gnu -lsci_gnu_mpi" FFTW3_LIBS="-L${FFTW_ROOT}/lib -lfftw3 -lfftw3f" ../configure --prefix=`pwd` --with-mpi=yes --enable-mpi-io=yes --enable-openmp=yes --with-linalg-flavor=netlib+elpa --with-fft-flavor=fftw3 --with-elpa=${EBROOTELPA} --with-libxc=${EBROOTLIBXC} --enable-memory-profiling --enable-mpi-inplace=yes --with-optim-flavor=standard --enable-gw-dpc=yes
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
salloc -N 1 -c 128 --account=<your_project_number> -t 00:30:00 -p debug
../tests/runtests.py -n 1 -j 128 --use-srun --mpi-args="--cpus-per-task=1" -t 900
```

```plaintext
Suite            failed  passed  succeeded  skipped  disabled  run_etime  tot_etime
atdep                 0       0         38        0         0     910.34     917.85
atompaw               0       0          0        2         0       0.00       0.00
bigdft                0       0          0       19         0       0.00       0.03
bigdft_paral          0       0          0        4         0       0.00       0.01
built-in              0       0          5        2         0      26.10      26.15
etsf_io               0       0          8        0         0      56.99      57.36
fast                  0       1         10        0         0      88.03      89.30
gpu                   0       0          0        8         0       0.00       0.01
gpu_kokkos            0       0          0        1         0       0.00       0.00
gpu_omp               0       0          0       50         0       0.00       0.15
gwpt_suite            0       0          0        2         0       0.00       0.01
gwr_suite             0       6          6        0         0     136.04     137.00
hpc_gpu_omp           0       0          0       13         0       0.00       0.02
libxc                 0       5         32        0         0     587.18     590.97
mpiio                 0       0          1       16         0       4.77       4.97
paral                 0       5         27      144         0     570.17     573.95
psml                  0       0          0       14         0       0.00       0.02
rttddft_suite         0       0          6        0         0      96.55      97.14
seq                   0       0          0       18         0       0.00       0.02
tutoatdep             0       0          5        0         0      27.24      27.66
tutomultibinit        0       1          5        4         0     324.62     328.87
tutoparal             0       0          4       24         0     136.24     136.55
tutoplugs             0       0          0        8         0       0.00       0.01
tutorespfn            0      12         17        4         0    5556.80    5563.32
tutorial              0      14         51        0         0    3760.83    3766.55
unitary               0       3         28        7         0     221.60     222.41
v1                    0       2         72        0         0     758.98     763.47
v10                   0       9         29        6         0    1664.04    1667.17
v2                    0      15         64        0         0    3053.61    3059.62
v3                    0      15         63        0         0   21431.71   21439.79
v4                    0      14         47        0         0   17652.72   17660.31
v5                    0      11         61        0         0   11074.08   11082.29
v6                    0      10         50        0         0    5818.95    5827.56
v67mbpt               0       3         25        0         0    3759.20    3765.48
v7                    0      18         47        0         0    5627.01    5637.90
v8                    1      10         55        5         0    4895.13    4907.25
v9                    0      16         90        2         0    4573.84    4589.25
vdwxc                 0       0          0        1         0       0.00       0.00
wannier90             0       0          0       10         0       0.00       0.01
```

All failed tests are due to known memory leakage.

```bash
../tests/runtests.py -n 4 -j 32 --use-srun --mpi-args="--cpus-per-task=1" -t 900
```

```plaintext
Suite            failed  passed  succeeded  skipped  disabled  run_etime  tot_etime
atdep                 0       0         38        0         0     570.22     578.97
atompaw               0       0          0        2         0       0.00       0.00
bigdft                0       0          0       19         0       0.00       0.02
bigdft_paral          0       0          0        4         0       0.00       0.00
built-in              0       0          5        2         0      14.05      14.12
etsf_io               0       0          8        0         0      28.37      28.80
fast                  0       1         10        0         0      48.64      50.22
gpu                   0       0          0        8         0       0.00       0.01
gpu_kokkos            0       0          0        1         0       0.00       0.00
gpu_omp               0       0          0       50         0       0.00       0.06
gwpt_suite            0       0          0        2         0       0.00       0.01
gwr_suite             0      11          1        0         0      64.02      65.00
hpc_gpu_omp           0       0          0       13         0       0.00       0.01
libxc                 0       5         32        0         0     358.58     362.42
mpiio                 0       1         12        4         0     106.40     112.42
paral                 1      12         55      108         0     581.08     602.75
psml                  0       0          0       14         0       0.00       0.02
rttddft_suite         0       0          6        0         0      46.20      46.86
seq                   0       0          0       18         0       0.00       0.02
tutoatdep             0       0          5        0         0      11.68      12.52
tutomultibinit        0       1          5        4         0     115.88     120.60
tutoparal             0       1          0       27         0      29.91      30.60
tutoplugs             0       0          0        8         0       0.00       0.01
tutorespfn            0      12         17        4         0    1108.72    1115.27
tutorial              2      11         52        0         0     993.54    1000.21
unitary               1       3         27        7         0     282.95     284.03
v1                    0       2         72        0         0    4893.32    4899.46
v10                   2       7         29        6         0    1467.46    1471.24
v2                    0      16         63        0         0    4626.31    4633.19
v3                    0      14         64        0         0    3434.05    3443.66
v4                    1      14         46        0         0    3226.57    3234.12
v5                    2      10         60        0         0    2121.51    2129.27
v6                    0      10         50        0         0    1417.22    1425.64
v67mbpt               1       3         24        0         0     731.92     737.89
v7                    2      14         49        0         0    2237.66    2248.69
v8                    2      11         53        5         0    2400.75    2412.35
v9                    2      16         88        2         0    2395.70    2410.30
vdwxc                 0       0          0        1         0       0.00       0.00
wannier90             0       0          0       10         0       0.00       0.01
```

![image](/assets/images/image-20260217091452-0i90isi.png)
