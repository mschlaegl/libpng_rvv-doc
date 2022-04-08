# [libpng_rvv](https://github.com/mschlaegl/libpng_rvv): A RISC-V Vector Optimized libpng

(C) 2022 Manfred SCHLÄGL <manfred.schlaegl@gmx.at>

**In this article, we present [libpng_rvv](https://github.com/mschlaegl/libpng_rvv):
A customized variant of [libpng](https://github.com/glennrp/libpng), which was optimized
for RISC-V vector extension (RVV) using [RVVRadar](https://github.com/mschlaegl/RVVRadar)**

Libpng already provides infrastructure to support architecture specific implementations of filter types to improve performance.
This includes implementations for Intel SSE2, ARM Neon, MIPS MSA, and PowerPC VSX which can be enabled at build time or in some cases automatically at run-time.
The implementation presented here uses the same infrastructure to add support for, and improve performance on RISC-V Cores implementing RVV.

The implementation provides a speedup of filter calculations on decompression of up to 5.43 (on Allwinner D1).
It supports RVV drafts 0.7.1, 0.8, 0.9, 0.10 and release 1.0, and is available on github [libpng_rvv](https://github.com/mschlaegl/libpng_rvv). Furthermore the modification was also submitted to the *png-mgn-implement* mailing list.


The work was mainly done in the context of a bachelor theses at the Institute
for Complex Systems (ICS), JKU Linz. With
[RVVRadar: A Framework for Supporting the Programmer in Vectorization for RISC-V](https://www.ics.jku.at/files/2022GLSVLSI_RVVRadar.pdf)
there is also a paper resulting from this work. A BibTex entry to cite to the
paper can be found in the last section.
Special thanks to Dr. Daniel Große and Lucas Klemmer, MSc for advise and
mentoring.

In the first part we describe the development and evaluation process using [RVVRadar](https://github.com/mschlaegl/RVVRadar). We then show how the evaluation can be reproduced and how [libpng_rvv](https://github.com/mschlaegl/libpng_rvv) can be built for hardware implementing one of the supported RVV versions.



## Development and Evaluation

Development and evaluation was done using [RVVRadar](https://github.com/mschlaegl/RVVRadar) which was also developed in context of the same bachelor theses.

[RVVRadar](https://github.com/mschlaegl/RVVRadar) is an open source framework to support the programmer over the four major steps of
development, verification, measurement and evaluation during the vectorization
process of algorithms for RVV.

It follows a bottom-up design approach. When a new algorithm has to be
implemented for some target software system (like libpng), it is added to the framework
including a baseline implementation. In an iterative manner, programmers can
add new optimized implementations to [RVVRadar](https://github.com/mschlaegl/RVVRadar). When the framework is executed
on a target hardware platform it automatically verifies and measures the
performance of all implementations and generates standardized statistics.
Based on these statistics the implementations can be optimized or a completely
new implementation can be added.

In case of the PNG filter types, we first integrated baseline C implementations for *Up*, *Avg*, *Sub*, and *Paeth* filter types in [RVVRadar](https://github.com/mschlaegl/RVVRadar).
After that, in multiple iterations we added, verified, evaluated and optimized RVV implementations for the filter types.
Details on PNG and the filter types can be found in the [PNG Specification - Section 9](https://www.w3.org/TR/PNG/#9Filters).

In a final step, we selected the best performing implementations and integrated them in libpng.
In the next sections we present the final performance evaluations and implementations.


### Experimental Setup

Since RVV is still in ratification, currently RISC-V hardware capable
of running Linux and supporting RVV is hardly available.
However, one such board is the very recently released Nezha from RVBoards
with an Allwinner D1 SoC.

 * Hardware: [RVBoards Nezha](https://linux-sunxi.org/Allwinner_Nezha)
   * SoC: Allwinner D1
   * Core: Xuantie C906 RISC-V
     * RV64GCV
     * RVV Draft 0.7.1
 * Software:
   * Linux (kernel): 5.4.61 from original BSP (including Vector Support)
   * Toolchain and Rootfs: [rvv_buildroot_build/rvv-0.7.1](https://github.com/mschlaegl/rvv_buildroot_build/tree/rvv-0.7.1) which uses:
     * [riscv-gnu-toolchain/rvv-0.7.1](https://github.com/riscv-collab/riscv-gnu-toolchain/tree/rvv-0.7.1)
     * [buildroot-2021.08](https://git.buildroot.net/buildroot/tree/?h=2021.08)

([see below](#reproducing-results) for details on how to reproduce the measurements)

To use [RVVRadar](https://github.com/mschlaegl/RVVRadar) it is necessary to specify a range of workloads
to be processed by the implementations being measured. A workload
corresponds to the number of elements processed by the implementations
which in case of the filter methods is the number of pixels. Since
all filter type algorithms have linear run-time complexity wrt. pixels
times the constant number color channels, measurements on a single
workload are sufficient for performance evaluation.

Each run-time measurement has an overhead caused by the instrumentation
for the measurement itself. To minimize the impact of
this overhead, the workload must be chosen such that the amount of
time spend on running an implementation is much larger than the
amount of time spent on the measurement. To fulfill this requirement
we selected a single workload of 50 million elements (50 megapixels).

[RVVRadar](https://github.com/mschlaegl/RVVRadar) provides extensive statistics for each algorithm implementation,
which allows for detailed analysis:
 * the minimum/maximum run-time,
 * the arithmetic mean run-time (including variance and standard deviation),
 * the median run-time
 * and a run-time histogram with 20 buckets between min and max run-time

For performance evaluation the average run-times are the most
interesting values. [RVVRadar](https://github.com/mschlaegl/RVVRadar) provides two averaging methods for its run-time
measurements, namely the arithmetic mean and the median. Since GNU/Linux is
not a hard real-time system and thus not deterministic there are run-time
spikes. For this reason, we used the median, as it is more robust against such outliers.
The average throughput is then simply calculated by dividing the number of elements by the median run-time.

Additional diagrams based on the min(best) and mean run-time can be found in [figures/rvvradar_c8dd25d911_q_r0_i100_s50000000_e50000000_all/results](https://github.com/mschlaegl/libpng_rvv-doc/tree/main/figures/rvvradar_c8dd25d911_q_r0_i100_s50000000_e50000000_all/results) and the raw results of [RVVRadar](https://github.com/mschlaegl/RVVRadar) are available in [rvvradar_c8dd25d911_q_r0_i100_s50000000_e50000000_all.csv](https://github.com/mschlaegl/libpng_rvv-doc/blob/main/figures/rvvradar_c8dd25d911_q_r0_i100_s50000000_e50000000_all.csv)

There are two result diagrams for each PNG filter type. One for 3 color channels per pixel (RGB) and one for 4 color channels per pixel (RGBA). So the last number (*up3*, *up4*, ...) corresponds to the number of color channels per pixel.

The X-axis of the diagrams shows the number of elements corresponding to pixels processed in one iteration and the Y-axis shows the median throughput in million elements per second which corresponds to megapixels per second. The value over each bar shows the exact throughput in megapixels per second. The different implementations are color-coded and listed in order of development in the legend on the left.

The *c byte noavect* and *c byte avect* implementations correspond to the C implementation compiled with GCC *-O3* and without and with using GCCs auto-vectorization (*--ftree-vectorize*), respectively. Note that GCCs auto-vectorizer does not yet use RVV, but there is some vectorization which works using integer registers and instructions only.
The implementations starting with *rvv* are the RVV implementations which will be discussed in each section below.

### PNG Up

<img src="figures/rvvradar_c8dd25d911_q_r0_i100_s50000000_e50000000_all/results/png_filters_up3/median_%5BMEps%5D.png" width=70%/>

<img src="figures/rvvradar_c8dd25d911_q_r0_i100_s50000000_e50000000_all/results/png_filters_up4/median_%5BMEps%5D.png" width=70%/>

 * **Note:** The result of a pixel does not depend on previous pixel -> Parallelization is restricted only by number of pixels in a row.
 * Baseline C Implementations [RVVRadar-0.8/algorithms/png_filters/up_impl_c.c.in](https://github.com/mschlaegl/RVVRadar/blob/RVVRadar-0.8/algorithms/png_filters/up_impl_c.c.in)
   * *c byte noavect*
   * *c byte avect* (optimization possible -> add of 8 elements in single 64bit integer register)
 * RVV Implementations [RVVRadar-0.8/algorithms/png_filters/up_impl_rvv.c](https://github.com/mschlaegl/RVVRadar/blob/RVVRadar-0.8/algorithms/png_filters/up_impl_rvv.c)
   * *rvv_m1* .. uses only one vector register (128bit) per loop iteration
   * *rvv_m2* .. uses vector register grouping of two registers (2\*128bit) per loop iteration
   * *rvv_m4* .. uses vector register grouping of four registers (4\*128bit) per loop iteration
   * *rvv_m8* .. uses vector register grouping of eight registers (8\*128bit) per loop iteration **-> Selected for integration**

### PNG Sub
<img src="figures/rvvradar_c8dd25d911_q_r0_i100_s50000000_e50000000_all/results/png_filters_sub3/median_%5BMEps%5D.png" width=70%/>

<img src="figures/rvvradar_c8dd25d911_q_r0_i100_s50000000_e50000000_all/results/png_filters_sub4/median_%5BMEps%5D.png" width=70%/>

 * **Note:** The result of a pixel depends on previous pixel -> Parallelization is restricted to number of colors per pixel (RGB=3, RGBA=4). This is also the reason why the performance gain is higher for RGBA(4).
 * Baseline C Implementations [RVVRadar-0.8/algorithms/png_filters/sub_impl_c.c.in](https://github.com/mschlaegl/RVVRadar/blob/RVVRadar-0.8/algorithms/png_filters/sub_impl_c.c.in)
   * *c byte noavect*
   * *c byte avect* (optimization not possible)
 * RVV Implementations [RVVRadar-0.8/algorithms/png_filters/sub_impl_rvv.c](https://github.com/mschlaegl/RVVRadar/blob/RVVRadar-0.8/algorithms/png_filters/sub_impl_rvv.c)
   * *rvv_dload* .. reloads the calculated pixel in next iteration
   * *rvv_reuse* .. reuses the calculated pixel in next iteration **-> Selected for integration**

### PNG Avg
<img src="figures/rvvradar_c8dd25d911_q_r0_i100_s50000000_e50000000_all/results/png_filters_avg3/median_%5BMEps%5D.png" width=70%/>

<img src="figures/rvvradar_c8dd25d911_q_r0_i100_s50000000_e50000000_all/results/png_filters_avg4/median_%5BMEps%5D.png" width=70%/>

 * **Note:** The result of a pixel depends on previous pixel -> Parallelization is restricted to number of colors per pixel (RGB=3, RGBA=4). This is also the reason why the performance gain is higher for RGBA(4).
 * Baseline C Implementations [RVVRadar-0.8/algorithms/png_filters/avg_impl_c.c.in](https://github.com/mschlaegl/RVVRadar/blob/RVVRadar-0.8/algorithms/png_filters/avg_impl_c.c.in)
   * *c byte noavect*
   * *c byte avect* (optimization not possible)
 * RVV Implementations [RVVRadar-0.8/algorithms/png_filters/avg_impl_rvv.c](https://github.com/mschlaegl/RVVRadar/blob/RVVRadar-0.8/algorithms/png_filters/avg_impl_rvv.c)
   * *rvv* **-> Selected for integration**


### PNG Paeth
<img src="figures/rvvradar_c8dd25d911_q_r0_i100_s50000000_e50000000_all/results/png_filters_paeth3/median_%5BMEps%5D.png" width=70%/>

<img src="figures/rvvradar_c8dd25d911_q_r0_i100_s50000000_e50000000_all/results/png_filters_paeth4/median_%5BMEps%5D.png" width=70%/>

 * **Note:** The result of a pixel depends on previous pixel -> Parallelization is restricted to number of colors per pixel (RGB=3, RGBA=4). This is also the reason why the performance gain is higher for RGBA(4).
 * Baseline C Implementations [RVVRadar-0.8/algorithms/png_filters/paeth_impl_c.c.in](https://github.com/mschlaegl/RVVRadar/blob/RVVRadar-0.8/algorithms/png_filters/paeth_impl_c.c.in)
   * *c byte noavect*
   * *c byte avect* (optimization not possible)
 * RVV Implementations
   * *rvv* [RVVRadar-0.8/algorithms/png_filters/paeth_impl_rvv.c](https://github.com/mschlaegl/RVVRadar/blob/RVVRadar-0.8/algorithms/png_filters/paeth_impl_rvv.c) .. loads and calculates one pixel per iteration **-> Selected for integration**
   * *rvv bulk load* [RVVRadar-0.8/algorithms/png_filters/paeth_impl_rvv_bulk_load.c](https://github.com/mschlaegl/RVVRadar/blob/RVVRadar-0.8/algorithms/png_filters/paeth_impl_rvv_bulk_load.c) .. loads maximum number of pixels in a vector register and processes them one after another (using `vslidedown.vx`). The Performance is worse than *rvv*. The overhead of the additional nested loop seems to outweigh the gain of loading more elements at once.

### Summary

Speedups relative to the baseline C implementation compiled without using GCCs auto-vectorizer (*c byte noavect*)

| **Algorithm**       | **Speedup** |
| ------------------- | ----------- |
| png_filters_up3     | 5.43        |
| png_filters_up4     | 5.43        |
| png_filters_sub3    | 2.19        |
| png_filters_sub4    | 3.07        |
| png_filters_avg3    | 1.55        |
| png_filters_avg4    | 2.07        |
| png_filters_paeth3  | 1.13        |
| png_filters_paeth4  | 1.51        |

## Reproducing Results

In this section we provide some informations on how to reproduce above measurements on RISC-V cores which implement different RVV versions.

*Running tests on similar or different hardware and sharing the results is greatly appreciated.*

### Preparations

(At time of writing RVV was neither supported by vanilla GCC nor Linux)

 1. Determine the RVV version implemented by your hardware (SoC/Core datasheet)
    * [RVVRadar](https://github.com/mschlaegl/RVVRadar) supports drafts 0.7.1, 0.8, 0.9, 0.10 and release 1.0
 1. Get a toolchain for your RVV version:
    * Use the toolchain + rootfs bundle provided by us [rvv_buildroot_build](https://github.com/mschlaegl/rvv_buildroot_build)
    * Or use [riscv-gnu-toolchain](https://github.com/riscv-collab/riscv-gnu-toolchain) directly (not covered here)
    * In both cases the branches correspond to following RVV versions:
      * *basic-rvv* .. RVV 1.0
      * *rvv-intrinsic* .. RVV 0.10
      * *rvv-0.9.x* .. RVV 0.9
      * *rvv-0.8.x* .. RVV 0.8
      * *rvv-0.7.1* .. RVV 0.7.1
 1. Get a Linux kernel with applied RVV patchset for your RVV version and enable support for RVV
    * e.g. RVV Patchset V9 for linux-5.15: [rvv_buildroot_build/tree/basic-rvv/PATCHES/linux-5.15/vector_v9](https://github.com/mschlaegl/rvv_buildroot_build/tree/basic-rvv/PATCHES/linux-5.15/vector_v9)
    * Config: ```CONFIG_VECTOR=y```

### Getting and Building Toolchain and Rootfs
(using [rvv_buildroot_build](https://github.com/mschlaegl/rvv_buildroot_build))

 * Ensure that your system fulfills the requirements of buildroot: [Buildroot Manual -- Chapter 2. System requirements](https://buildroot.org/downloads/manual/manual.html#requirement)
 * Ensure that you have at least 25GB of free memory
 * Clone the branch corresponding to your RVV version from [rvv_buildroot_build](https://github.com/mschlaegl/rvv_buildroot_build). e.g. for RVV 0.7.1
```
$ git clone https://github.com/mschlaegl/rvv_buildroot_build.git -b rvv-0.7.1 --recursive
```
 * Build toolchain and rootfs
```
$ cd rvv_buildroot_build
$ ./build.sh
```
   * Toolchain and rootfs are now located in ```output```

More information on buildroot can be found at [buildroot.org](https://buildroot.org) and in the [Buildroot Manual](https://buildroot.org/downloads/manual/manual.html).

### Setting Up a Running System
This is very specific to your hardware, BSP and used environment and can therefore not described in detail here.

You have to ensure that your board is running
 * a RVV capable Linux kernel (see [Preparations](#preparations)) and
 * a rootfs compatible to the generated toolchain. (e.g. created in [Getting and Building Toolchain and Rootfs](#getting-and-building-toolchain-and-rootfs).)

### Getting, Building and Installing [RVVRadar](https://github.com/mschlaegl/RVVRadar)
 * Clone the latest version of [RVVRadar](https://github.com/mschlaegl/RVVRadar). e.g. RVVRadar-0.8
```
$ git clone https://github.com/mschlaegl/RVVRadar.git -b RVVRadar-0.8
```
 * Change into directory *rvvradar*
 * Setup the running shell for cross build <br> (```<rvv_buildroot_build>``` .. path to ```rvv_buildroot_build``` from [Getting and Building Toolchain and Rootfs](#getting-and-building-toolchain-and-rootfs))
```
$ . <rvv_buildroot_build>/output/host/environment-setup
```
 * Configure [RVVRadar](https://github.com/mschlaegl/RVVRadar)
```
$ ./configure
```
 * For RVV 0.7.1, the output of configure should contain ```Check toolchain RISC-V vector support .. yes (rvv v0.7)```
 * Build
```
$ make
```
 * Install: Copy the binary *RVVRadar* to the target rootfs

More information can be found in the [RVVRadar readme](https://github.com/mschlaegl/RVVRadar/blob/master/README.md).

### Executing [RVVRadar](https://github.com/mschlaegl/RVVRadar)

#### Test/Verification Run
Test [RVVRadar](https://github.com/mschlaegl/RVVRadar) and verify included implementations.

Run a single iteration of all algorithm implementations with 256 elements and check output for ```fails```
```
$ RVVRadar --algs_enabled 0x7FF --len_start 256 --iterations 1 --verify
```

#### Evaluation Run
Run 100 iterations of all algorithms implementations with 50 million elements and save machine processable output to results.csv
```
$ RVVRadar --randseed 0 --algs_enabled 0x7FF --len_start 50000000 --iterations 100 --quiet > results.csv
```

(As described [above](#experimental-setup) use the *tdmedian [ns]* values and divide them by number of elements (50 million) to get the average throughputs.)


## Building [libpng_rvv](https://github.com/mschlaegl/libpng_rvv)
To be able to build [libpng_rvv](https://github.com/mschlaegl/libpng_rvv) you have to do the [preparations from above](#preparations) and have a working [toolchain for your specific RVV version](#get-and-build-toolchain-and-rootfs).

 * Clone the latest version of [libpng_rvv](https://github.com/mschlaegl/libpng_rvv)
```
$ git clone https://github.com/mschlaegl/libpng_rvv.git -b libpng16_rvv
```
 * Change into directory *libpng_rvv*
 * Setup the running shell for cross build <br> (```<rvv_buildroot_build>``` .. path to ```rvv_buildroot_build``` from [Get and Build Toolchain and Rootfs](#getting-and-building-toolchain-and-rootfs))
```
$ . <rvv_buildroot_build>/output/host/environment-setup
```
 * Enable build for RVV by adding *v* to *--march*. e.g. for RV64GC
```
$ export CFLAGS="$CFLAGS --march=rv64gcv"
```
 * Configure [libpng_rvv](https://github.com/mschlaegl/libpng_rvv) for (unconditional) use of RVV
   * RVV 1.0: <br>```$ ./configure --host=riscv64-unknown-linux --enable-riscv-vector=yes```
   * RVV 0.10: <br>```$ ./configure --host=riscv64-unknown-linux --enable-riscv-vector=yes --enable-riscv-vector-compat=0.10```
   * RVV 0.9: <br>```$ ./configure --host=riscv64-unknown-linux --enable-riscv-vector=yes --enable-riscv-vector-compat=0.9```
   * RVV 0.8: <br>```$ ./configure --host=riscv64-unknown-linux --enable-riscv-vector=yes --enable-riscv-vector-compat=0.8```
   * RVV 0.7.1: <br>```$ ./configure --host=riscv64-unknown-linux --enable-riscv-vector=yes --enable-riscv-vector-compat=0.7.1```
 * Build
```
make
```




## Additional Notes
 * [RVVRadar](https://github.com/mschlaegl/RVVRadar) contains not only PNG filter types, but also some other, more general algorithms. Since they are not part of the topic, they are not covered here. However, the result diagrams for them are available in [figures/rvvradar_c8dd25d911_q_r0_i100_s50000000_e50000000_all/results](https://github.com/mschlaegl/libpng_rvv-doc/tree/main/figures/rvvradar_c8dd25d911_q_r0_i100_s50000000_e50000000_all/results) and the raw results in [rvvradar_c8dd25d911_q_r0_i100_s50000000_e50000000_all.csv](https://github.com/mschlaegl/libpng_rvv-doc/blob/main/figures/rvvradar_c8dd25d911_q_r0_i100_s50000000_e50000000_all.csv) also contain them.


## *RVVRadar: A Framework for Supporting the Programmer in Vectorization for RISC-V*

[Lucas Klemmer, Manfred Schlaegl, and Daniel Große. RVVRadar: a framework for supporting the programmer in vectorization for RISC-V. In GLSVLSI, 2022](https://www.ics.jku.at/files/2022GLSVLSI_RVVRadar.pdf)

```
@InProceedings{KSG:2022,
  author        = {Lucas Klemmer and Manfred Schlaegl and Daniel Gro{\ss}e},
  title         = {{RVVRadar:} A Framework for Supporting the Programmer in Vectorization for {RISC-V}},
  booktitle     = {ACM Great Lakes Symposium on VLSI},
  year          = 2022
}
```
