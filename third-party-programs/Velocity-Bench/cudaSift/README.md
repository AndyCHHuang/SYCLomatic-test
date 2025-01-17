# Migration example: Migrate cudaSift to SYCL version
[SYCLomatic](https://github.com/oneapi-src/SYCLomatic) is a project to assist developers in migrating their existing code written in different programming languages to the SYCL* C++ heterogeneous programming model. It is open source version of Intel® DPC++ Compatibility Tool.

This file lists the detail steps to migrate CUDA version of [cudaSift](https://github.com/oneapi-src/Velocity-Bench/tree/main/cudaSift) to SYCL version with SYCLomatic. As follow table summaries the migration environment, software required and so on.

   | Optimized for         | Description
   |:---                   |:---
   | OS                    | Linux* Ubuntu* 22.04
   | Software              | Intel® oneAPI Base Toolkit, SYCLomatic
   | What you will learn   | Migration of CUDA code, Run SYCL code on oneAPI and Intel device
   | Time to complete      | 15 minutes


## Migrating cudaSift to SYCL

### 1 Prepare the migration
#### 1.1 Get the source code of cudaSift and install the dependency library
```sh
   $ git clone https://github.com/oneapi-src/Velocity-Bench.git
   $ export cudaSift_HOME=/path/to/Velocity-Bench/cudaSift
   $ sudo apt-get install libopencv-dev # make sure ```OpenCV*``` is installed on the machine.
   $ cd ${cudaSift_HOME}/CUDA && mkdir build
   $ cd build && cmake -DUSE_SM=80 ..   # make sure all dependency library are installed.
```
Summary of cudaSift project source code:  12 files in CUDA folder

   ```
      CUDA
      ├── CMakeLists.txt
      ├── cudaImage.cu
      ├── cudaImage.h
      ├── cudaSift.h
      ├── cudaSiftD.cu
      ├── cudaSiftD.h
      ├── cudaSiftH.cu
      ├── cudaSiftH.h
      ├── cudautils.h
      ├── geomFuncs.cpp
      ├── mainSift.cpp
      └── matching.cu
   ```
#### 1.2 Prepare migration tool and SYCL run environment

 * Install SYCL run environment [Intel® oneAPI Base Toolkit](https://www.intel.com/content/www/us/en/developer/tools/oneapi/base-toolkit.html). After install, Intel® DPC++ Compatibility tool is also availalbe, setup the SYCL run environment as follow:

```
   $ source /opt/intel/oneapi/setvars.sh
   $ dpct --version  # Intel® DPC++ Compatibility tool version
```
 * If want to try latest version of compatibility tool, try to install SYCLomatic by download prebuild of [SYCLomatic release](https://github.com/oneapi-src/SYCLomatic/blob/SYCLomatic/README.md#Releases) or [build from source](https://github.com/oneapi-src/SYCLomatic/blob/SYCLomatic/README.md), as follow give the steps to install prebuild version: 
 ```
   $ export SYCLomatic_HOME=/path/to/install/SYCLomatic
   $ mkdir $SYCLomatic_HOME
   $ cd $SYCLomatic_HOME
   $ wget https://github.com/oneapi-src/SYCLomatic/releases/download/20240203/linux_release.tgz   #Change the timestamp 20240203 to latest one
   $ tar xzvf linux_release.tgz
   $ source setvars.sh
   $ dpct --version #SYCLomatic version
 ```
 
 
For more information on configuring environment variables, see [Use the setvars Script with Linux*](https://www.intel.com/content/www/us/en/develop/documentation/oneapi-programming-guide/top/oneapi-development-environment-setup/use-the-setvars-script-with-linux-or-macos.html).
 
### 2 Generate the compilation database 
``` sh
$ cd ${cudaSift_HOME}/CUDA/build
$ make clean
$ intercept-build make
$ ls compile_commands.json  # make sure compile_commands.json is generated
compile_commands.json
```
### 3 Migrate the source code and build script
**Follow section 3.1 or section 3.2 to migrate the source code and corresponding build scripts.**
## 3.1 Migrate the source code and generate build script `Makefile.dpct`
```sh
# From the CUDA directory as root directory:
$ cd ${cudaSift_HOME}/CUDA
$ dpct --in-root=. -p=./build/compile_commands.json --out-root=out --gen-build-script --cuda-include-path=/usr/local/cuda/include
```
Description of the options: 
 * `--in-root`: provide input files to specify where to locate the CUDA files that needs migration.
 * `-p`: specify compilation database to migrate the whole project.
 * `--out-root`: designate where to generate the resulting files (default is `dpct_output`).
 * `--gen-build-script`: generate the `Makefile.dpct` for the migrated code.

Now you can see the migrated files in the `out` folder as follow: 
   ```
      out/
      ├── MainSourceFiles.yaml
      ├── cudaImage.dp.cpp
      ├── cudaImage.h
      ├── cudaSift.h
      ├── cudaSift.h.yaml
      ├── cudaSiftD.dp.cpp
      ├── cudaSiftD.h
      ├── cudaSiftH.dp.cpp
      ├── cudaSiftH.h
      ├── cudautils.h
      ├── cudautils.h.yaml
      ├── geomFuncs.cpp
      ├── mainSift.cpp.dp.cpp
      └── matching.dp.cpp

   ```

## 3.2 Migrate the source code and CMake build script
### 3.2.1 Migrate the source code
Run the migration command below to generate SYCL version cudaSift code:
```sh
$ dpct --in-root=. -p=./build/compile_commands.json --out-root=out --gen-build-script --cuda-include-path=/usr/local/cuda/include
```
### 3.2.2 Migrated the CMake build script
Apply option "--migrate-build-script-only" to the migration command as follows to migrate the CMake build script:
```
$ dpct --in-root=. -p=./build/compile_commands.json --out-root=out --gen-build-script --cuda-include-path=/usr/local/cuda/include --migrate-build-script-only
```
Or, CMake script can be migrated together with cudaSift source code with the option "--migrate-build-script=CMake" as follows:
```sh
$ dpct --in-root=. -p=./build/compile_commands.json --out-root=out --gen-build-script --cuda-include-path=/usr/local/cuda/include --migrate-build-script=CMake
```
### 3.2.3 Apply necessary manual fix to SYCL version CMake build script
Do the following changes to the migrated CMake script in the `out` directory:
```
diff --git /path/to/cuda/CMakeLists.txt /path/to/SYCL/CMakeLists.txt
index d1e4b6d..ba85f53 100644
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -56,13 +56,13 @@ set(cuda_sources
 )

 set(sources
-  ${CMAKE_SOURCE_DIR}/../common/Utility.cpp
+  ${CMAKE_SOURCE_DIR}/common/Utility.cpp
   geomFuncs.cpp
   mainSift.cpp.dp.cpp
 )

 include_directories(
-  ${CMAKE_SOURCE_DIR}/../common/
+  ${CMAKE_SOURCE_DIR}/common/
   ${CMAKE_CURRENT_SOURCE_DIR}
 )

```
Also copy the `common` directory to the `out` directory, using the following commands:
```sh
$cd ${cudaSift_HOME}/CUDA/out
$cp ../../common/ ./ -r
```
### 3.2.4 Build the SYCL version cudaSift code with CMake build script
In the `out` directory, use the following commands to build the SYCL version cudaSift:
```sh
$mkdir -p build
$cd build
$cmake -DCMAKE_C_COMPILER=icx    -DCMAKE_CXX_COMPILER=icpx -DUSE_SM=80 -DCMAKE_VERBOSE_MAKEFILE:BOOL=ON ..
$make
```
After build, the binary `cudasift` is generated in the directory `${cudaSift_HOME}/CUDA/out/build`
### 3.2.5 Run the SYCL version cudaSift
As the path of test data is relative to the path of CUDA version binary, the SYCL version binary should be moved to the same level directory where the CUDA version binary lies.
Run the following commands to run the SYCL version binary.
```
$cp cudasift ../../build/cudasift_sycl.run
$cd ../../build
$./cudasift_sycl.run
```


### 4 Review the migrated source code and fix all `DPCT` warnings

SYCLomatic and [Intel® DPC++ Compatibility Tool](https://www.intel.com/content/www/us/en/developer/tools/oneapi/dpc-compatibility-tool.html) define a list of `DPCT` warnings and embed the warning in migrated source code if need manual effort to check. All the warnings in the migrated code should be reviewed and fixed. For detail of `DPCT` warnings and corresponding fix examples, refer to [Intel® DPC++ Compatibility Tool Developer Guide and Reference](https://www.intel.com/content/www/us/en/develop/documentation/intel-dpcpp-compatibility-tool-user-guide/top/diagnostics-reference.html) or [SYCLomatic doc page](https://oneapi-src.github.io/SYCLomatic/dev_guide/diagnostics-reference.html). 

Fix the warning in migrated cudaSift code:
```
$ cat ${cudaSift_HOME}/CUDA/out/Makefile.dpct
...
warning: #DPCT2001:228: You can link with more library by add them here.
LIB :=  
...
```
 **cudaSift** need to link the **OpenCV** libraries in link time, so fix LIB variable as follow:
```
LIB :=  -lopencv_core -lopencv_imgcodecs
```
### 5 Build the migrated cudaSift
```
$ cd ${cudaSift_HOME}/CUDA/out
$ make -f Makefile.dpct
```
### 6 Run migrated SYCL version cudaSift
```
   $ cd ${cudaSift_HOME}/CUDA/out
   $ ./cudasift 
   Image size = (1920,1080)
   Initializing data...
   Number of Points after sift extraction =  3681
   Number of Points after sift extraction =  3933
   Number of Points after sift extraction =  3681
   ...................
   Number of Points after sift extraction =  3933
   Number of original features: 3681 3933
   Number of matching features: 1220 1258 33.1432% 1 2

   Performing data verification 
   Data verification is SUCCESSFUL. 

   Total workload time = 2206.28 ms
```
**Note:** 
* The testing result was running on Intel(R) Core(TM) i7-13700K CPU backend with Intel® oneAPI Base Toolkit(2023.2 version).
* The Reference migrated code is attached in **migrated** folder.

If an error occurs during runtime, refer to [Diagnostics Utility for Intel® oneAPI Toolkits](https://www.intel.com/content/www/us/en/develop/documentation/diagnostic-utility-user-guide/top.html).

## cudaSift License
[License.txt](https://github.com/oneapi-src/Velocity-Bench/blob/main/cudaSift/LICENSE.md)

## Reference 
* Command Line Options of [SYCLomatic](https://oneapi-src.github.io/SYCLomatic/dev_guide/command-line-options-reference.html) or [Intel® DPC++ Compatibility Tool](https://software.intel.com/content/www/us/en/develop/documentation/intel-dpcpp-compatibility-tool-user-guide/top/command-line-options-reference.html)
* [oneAPI GPU Optimization Guide](https://www.intel.com/content/www/us/en/docs/oneapi/optimization-guide-gpu/)
* [SYCLomatic project](https://github.com/oneapi-src/SYCLomatic/)
* [Migrate a CMake Build Script](https://github.com/oneapi-src/SYCLomatic/blob/SYCLomatic/docs/dev_guide/migration/migrate-cmake-build.rst)

## Trademarks information
Intel, the Intel logo, and other Intel marks are trademarks of Intel Corporation or its subsidiaries.<br>
\*Other names and brands may be claimed as the property of others. SYCL is a trademark of the Khronos Group Inc.
