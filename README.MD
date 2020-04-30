[Using Microsoft Visual Studio 2019 Building A LLVM Out-Source-Tree Pass](https://llvm.org/docs/GettingStartedVS.html)
=========
  * [Development Environment](#development-environment)
  * [Building LLVM](#building-llvm)
  * [Building a Simple out-of-tree LLVM Pass](#building-a-simple-out-of-tree-llvm-pass)
  * [Testing a Simple out-of-tree LLVM Pass](#testing-a-simple-out-of-tree-llvm-pass)
  * [Debugging a Simple out-of-tree LLVM Pass](#debugging-a-simple-out-of-tree-llvm-pass)
  * [Common Problems](#common-problems)


Development Environment
==========
  * Windows 10 64bit
  * LLVM 10
  * Visual Studio 2019
  * [GnuWin32 tools](http://gnuwin32.sourceforge.net/)
  * [Python 2.7 or higher](http://www.python.org/)
  * [CMake 3.17.1 or higher](https://cmake.org/download/)

## Download LLVM 10.0.0源码
  * [LLVM source code](https://github.com/llvm/llvm-project/releases/download/llvmorg-10.0.0/llvm-10.0.0.src.tar.xz)
  * [Clang source code](https://github.com/llvm/llvm-project/releases/download/llvmorg-10.0.0/clang-10.0.0.src.tar.xz)
  * [LLD source code](https://github.com/llvm/llvm-project/releases/download/llvmorg-10.0.0/lld-10.0.0.src.tar.xz)
  * [LLDB source code](https://github.com/llvm/llvm-project/releases/download/llvmorg-10.0.0/lldb-10.0.0.src.tar.xz)
  * [Polly Source code](https://github.com/llvm/llvm-project/releases/download/llvmorg-10.0.0/polly-10.0.0.src.tar.xz)
  * [clang-tools-extra](https://github.com/llvm/llvm-project/releases/download/llvmorg-10.0.0/clang-tools-extra-10.0.0.src.tar.xz)
  * [compiler-rt source code](https://github.com/llvm/llvm-project/releases/download/llvmorg-10.0.0/compiler-rt-10.0.0.src.tar.xz)
  * [libc++ source code](https://github.com/llvm/llvm-project/releases/download/llvmorg-10.0.0/libcxx-10.0.0.src.tar.xz)
  * [libc++abi source code](https://github.com/llvm/llvm-project/releases/download/llvmorg-10.0.0/libcxxabi-10.0.0.src.tar.xz)
  * [libunwind source code](https://github.com/llvm/llvm-project/releases/download/llvmorg-10.0.0/libunwind-10.0.0.src.tar.xz)
  * [OpenMP Source code](https://github.com/llvm/llvm-project/releases/download/llvmorg-10.0.0/openmp-10.0.0.src.tar.xz)

有关更多详细信息，请阅读 [官方文件资料](https://llvm.org/docs/CMake.html).

## LLVM’s source tree
```
<llvm project source dir>/
    |
    <projects>/
        |
        <compiler-rt>
        <libcxx>
        <libcxxabi>
        <libunwind>
        <openmp>
        ...
    <lib>
        |
        ...
    <tools>
        |
        <clang>
            |
            <tools>
                |
                <clang-tools-extra>
                ...
            ...
        <lld>
        <lldb>
        <polly>
        ...
```

需要修改 **<<llvm project source dir>\include\llvm\Support\Compiler.h**, 使生成pass的dll能正常导出接口.
```C++
// FIXME: Provide this for PE/COFF targets.
#if (__has_attribute(weak) || LLVM_GNUC_PREREQ(4, 0, 0)) &&                    \
    (!defined(__MINGW32__) && !defined(__CYGWIN__) && !defined(_WIN32))
#define LLVM_ATTRIBUTE_WEAK __attribute__((__weak__))
#else
#define LLVM_ATTRIBUTE_WEAK __declspec(dllexport)
#endif
```

Building LLVM
=============
## using CMake (cmake-gui) Generate Visual Studio 2019 Project
![CMakeGui](https://github.com/Amoystyle/LLVM-VS/blob/master/CMakeGui.png)

设置以下几个值：
| Name                  | Vaule                   |
| --------------------- | :---------------------- |
| CMAKE_INSTALL_PREFIX  | K:\llvm\installed       |
| LLVM_TARGETS_TO_BUILD | all                     |
| LLVM_LIT_TOOLS_DIR    | K:\llvm\GetGnuWin32\bin |

点击**Configure**->**Generate**, 等待生成**Visual Studio 2019 Project**的项目(默认是x64工程), 如果想生成**x86**工程需要在**CMake GUI**配置**Configure**的时候配置项为**Win32**.

打开**Cmake Gui**配置生成项目**build**所在路径,双击打开**LLVM.sln**.
在Visual Studio 2019中开始编译LLVM，编译时间根据你PC的配置情况(**i7-8700k 16g**)前后编译时间长达**5个小时**，最后生成成功的**build目录**大小**(Debug)152G**.

生成成功,到此完成了LLVM的构建.


Building a Simple out-of-tree LLVM Pass
=======================================
以下代码为例：
```
<llvm-tutor source dir>/
    |
    CMakeLists.txt
    <HelloWorld>/
        |
        HelloWorld.cpp
        CMakeLists.txt
```

## CMakeLists.txt
```CMake
cmake_minimum_required(VERSION 3.4.3)
project(llvm-tutor)

#===============================================================================
# 1. VERIFY LLVM INSTALLATION DIR
#===============================================================================
# Set this to a valid LLVM installation dir
set(LT_LLVM_INSTALL_DIR "" CACHE PATH "LLVM installation directory")

# A bit of a sanity checking
set(LT_LLVM_INCLUDE_DIR "${LT_LLVM_INSTALL_DIR}/include/llvm")
if(NOT EXISTS "${LT_LLVM_INCLUDE_DIR}")
message(FATAL_ERROR
  " LT_LLVM_INSTALL_DIR (${LT_LLVM_INCLUDE_DIR}) is invalid.")
endif()

set(LT_LLVM_CMAKE_FILE "${LT_LLVM_INSTALL_DIR}/lib/cmake/llvm/LLVMConfig.cmake")
if(NOT EXISTS "${LT_LLVM_CMAKE_FILE}")
message(FATAL_ERROR
    " LT_LLVM_CMAKE_FILE (${LT_LLVM_CMAKE_FILE}) is invalid.")
endif()

#===============================================================================
# 2. LOAD LLVM CONFIGURATION
#    For more: http://llvm.org/docs/CMake.html#embedding-llvm-in-your-project
#===============================================================================
# Add the location of LLVMConfig.cmake to CMake search paths (so that
# find_package can locate it)
list(APPEND CMAKE_PREFIX_PATH "${LT_LLVM_INSTALL_DIR}/lib/cmake/llvm/")

find_package(LLVM 10.0.0 REQUIRED CONFIG)
message(STATUS "Found LLVM ${LLVM_PACKAGE_VERSION}")
message(STATUS "Using LLVMConfig.cmake in: ${LT_LLVM_INSTALL_DIR}")

message("LLVM STATUS:
  Definitions ${LLVM_DEFINITIONS}
  Includes    ${LLVM_INCLUDE_DIRS}
  Libraries   ${LLVM_LIBRARY_DIRS}
  Targets     ${LLVM_TARGETS_TO_BUILD}"
)

# Set the LLVM header and library paths
include_directories(${LLVM_INCLUDE_DIRS})
link_directories(${LLVM_LIBRARY_DIRS})
add_definitions(${LLVM_DEFINITIONS})

#===============================================================================
# 3. LLVM-TUTOR BUILD CONFIGURATION
#===============================================================================
# Use the same C++ standard as LLVM does
set(CMAKE_CXX_STANDARD 14 CACHE STRING "")

# Build type
if (NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Debug CACHE
      STRING "Build type (default Debug):" FORCE)
endif()
if(NOT MSVC)
    # Compiler flags
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall\
        -fdiagnostics-color=always")

    # LLVM is normally built without RTTI. Be consistent with that.
    if(NOT LLVM_ENABLE_RTTI)
      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-rtti")
    endif()

    # -fvisibility-inlines-hidden is set when building LLVM and on Darwin warnings
    # are triggered if llvm-tutor is built without this flag (though otherwise it
    # builds fine). For consistency, add it here too.
    include(CheckCXXCompilerFlag)
    check_cxx_compiler_flag("-fvisibility-inlines-hidden" SUPPORTS_FVISIBILITY_INLINES_HIDDEN_FLAG)
    if (${SUPPORTS_FVISIBILITY_INLINES_HIDDEN_FLAG} EQUAL "1")
      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fvisibility-inlines-hidden")
    endif()
endif()
# Set the build directories
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/bin")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/lib")

#===============================================================================
# 4. ADD SUB-TARGETS
# Doing this at the end so that all definitions and link/include paths are
# available for the sub-projects.
#===============================================================================

add_subdirectory(HelloWorld)
```

## HelloWorld/HelloWorld.cpp
```C++
//=============================================================================
// FILE:
//    HelloWorld.cpp
//
// DESCRIPTION:
//    Visits all functions in a module, prints their names and the number of
//    arguments via stderr. Strictly speaking, this is an analysis pass (i.e.
//    the functions are not modified). However, in order to keep things simple
//    there's no 'print' method here (every analysis pass should implement it).
//
// USAGE:
//    1. Legacy PM
//      opt -load libHelloWorld.dylib -legacy-hello-world -disable-output\
//        <input-llvm-file>
//    2. New PM
//      opt -load-pass-plugin=libHelloWorld.dylib -passes="hello-world"\
//        -disable-output <input-llvm-file>
//
//
// License: MIT
//=============================================================================
#include "llvm/IR/LegacyPassManager.h"
#include "llvm/Passes/PassBuilder.h"
#include "llvm/Passes/PassPlugin.h"
#include "llvm/Support/raw_ostream.h"

using namespace llvm;

//-----------------------------------------------------------------------------
// HelloWorld implementation
//-----------------------------------------------------------------------------
// No need to expose the internals of the pass to the outside world - keep
// everything in an anonymous namespace.
namespace {

// This method implements what the pass does
void visitor(Function &F) {
  errs() << "(llvm-tutor) Hello from: " << F.getName() << "\n";
  errs() << "(llvm-tutor)   number of arguments: " << F.arg_size() << "\n";
}

// New PM implementation
struct HelloWorld : PassInfoMixin<HelloWorld> {
  // Main entry point, takes IR unit to run the pass on (&F) and the
  // corresponding pass manager (to be queried if need be)
  PreservedAnalyses run(Function &F, FunctionAnalysisManager &) {
    visitor(F);
    return PreservedAnalyses::all();
  }
};

// Legacy PM implementation
struct LegacyHelloWorld : public FunctionPass {
  static char ID;
  LegacyHelloWorld() : FunctionPass(ID) {}
  // Main entry point - the name conveys what unit of IR this is to be run on.
  bool runOnFunction(Function &F) override {
    visitor(F);
    // Doesn't modify the input unit of IR, hence 'false'
    return false;
  }
};
} // namespace

//-----------------------------------------------------------------------------
// New PM Registration
//-----------------------------------------------------------------------------
llvm::PassPluginLibraryInfo getHelloWorldPluginInfo() {
  return {LLVM_PLUGIN_API_VERSION, "HelloWorld", LLVM_VERSION_STRING,
          [](PassBuilder &PB) {
            PB.registerPipelineParsingCallback(
                [](StringRef Name, FunctionPassManager &FPM,
                   ArrayRef<PassBuilder::PipelineElement>) {
                  if (Name == "hello-world") {
                    FPM.addPass(HelloWorld());
                    return true;
                  }
                  return false;
                });
          }};
}

// This is the core interface for pass plugins. It guarantees that 'opt' will
// be able to recognize HelloWorld when added to the pass pipeline on the
// command line, i.e. via '-passes=hello-world'
extern "C" LLVM_ATTRIBUTE_WEAK ::llvm::PassPluginLibraryInfo
llvmGetPassPluginInfo() {
  return getHelloWorldPluginInfo();
}

//-----------------------------------------------------------------------------
// Legacy PM Registration
//-----------------------------------------------------------------------------
// The address of this variable is used to uniquely identify the pass. The
// actual value doesn't matter.
char LegacyHelloWorld::ID = 0;

// This is the core interface for pass plugins. It guarantees that 'opt' will
// recognize LegacyHelloWorld when added to the pass pipeline on the command
// line, i.e.  via '--legacy-hello-world'
static RegisterPass<LegacyHelloWorld>
    X("legacy-hello-world", "Hello World Pass",
      true, // This pass doesn't modify the CFG => true
      false // This pass is not a pure analysis pass => false
    );

```

## HelloWorld/CMakeLists.txt
```CMake
cmake_minimum_required(VERSION 3.4.3)
project(llvm-tutor-hello-world)

#===============================================================================
# 1. LOAD LLVM CONFIGURATION
#===============================================================================
# Set this to a valid LLVM installation dir
set(LT_LLVM_INSTALL_DIR "" CACHE PATH "LLVM installation directory")

# Add the location of LLVMConfig.cmake to CMake search paths (so that
# find_package can locate it)
list(APPEND CMAKE_PREFIX_PATH "${LT_LLVM_INSTALL_DIR}/lib/cmake/llvm/")

find_package(LLVM 10.0.0 REQUIRED CONFIG)

# HelloWorld includes headers from LLVM - update the include paths accordingly
include_directories(${LLVM_INCLUDE_DIRS})

#===============================================================================
# 2. LLVM-TUTOR BUILD CONFIGURATION
#===============================================================================
# Use the same C++ standard as LLVM does
set(CMAKE_CXX_STANDARD 14 CACHE STRING "")

if(NOT MSVC)
    # LLVM is normally built without RTTI. Be consistent with that.
    if(NOT LLVM_ENABLE_RTTI)
      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-rtti")
    endif()
endif()
#===============================================================================
# 3. ADD THE TARGET
#===============================================================================
add_library(HelloWorld SHARED HelloWorld.cpp)

if(NOT MSVC)
    # Allow undefined symbols in shared objects on Darwin (this is the default
    # behaviour on Linux)
    target_link_libraries(HelloWorld
      "$<$<PLATFORM_ID:Darwin>:-undefined dynamic_lookup>")
else()
    llvm_map_components_to_libnames(llvm_libs Support Core IRReader TransformUtils)
    target_link_libraries(HelloWorld PRIVATE ${llvm_libs})
endif()
```

有关更多详细信息，请阅读 [官方文件资料](https://llvm.org/docs/CMake.html#embedding-llvm-in-your-project).

## using CMake (cmake-gui) Generate Visual Studio 2019 Project

设置以下几个值：
| Name                | Vaule         |
| ------------------- | :------------ |
| LT_LLVM_INSTALL_DIR | K:\llvm\build |

打开**Cmake Gui**配置生成项目**build**所在路径,双击打开**llvm-tutor.sln**编译,成功后在**build\bin\Debug**目录下生**HelloWorld.dll**

到此完成了LLVM Pass HelloWorld.dll的构建.

Testing a Simple out-of-tree LLVM Pass
======================================
### input_for_hello.c
```C++
long f(long a, long b)
{
    long x = 0;
    if (a > b) {
        x = 1;
    } 
    else {
        x = 2;
    }
    
    return x;
}

int main() {
    
    f(1, 2);

    return 0;
}
```

```bash
# Generate an LLVM test file
$LLVM_DIR/bin/clang -S -emit-llvm input_for_hello.c -o input_for_hello.ll
```
```bash
# Run the pass
$LLVM_DIR/bin/opt -load-pass-plugin HelloWorld.dll -passes=hello-world -disable-output input_for_hello.ll
# Expected output
(llvm-tutor) Hello from: f
(llvm-tutor)   number of arguments: 2
(llvm-tutor) Hello from: main
(llvm-tutor)   number of arguments: 0
```
HelloWorld.dll不会修改input_for_hello.ll模块,使用-disable-output是为了防止输出bitcode文件.

Debugging a Simple out-of-tree LLVM Pass
========================================
我们通过Visual Studio 2019来调试HelloWorld.dll的模块,方便了解LLVM Pass使用过程.
打开llvm-tutor.sln项目, 配置HelloWorld为**设为启动项目**

### 配置HelloWorld属性：
![HelloWorld](https://github.com/Amoystyle/LLVM-VS/blob/master/HelloWorld.png)

设置以下几个值：
| Name     | Vaule                                                                                         |
| -------- | :-------------------------------------------------------------------------------------------- |
| 命令     | $LLVM_DIR\bin\opt.exe                                                                         |
| 命令参数 | -load-pass-plugin HelloWorld.dll -passes=hello-world -S input_for_hello.ll -o output_hello.ll |

### 启动调试

![HelloWorldDebuging](https://github.com/Amoystyle/LLVM-VS/blob/master/HelloWorldDebuging.png)

### 生成HelloWorld.o：
Visual Studio 2019下需要使用`clang-cl.exe`来编译连接.
```bash
# Generate an LLVM IR file
$LLVM_DIR/bin/clang-cl.exe -Xclang -emit-llvm -c input_for_hello.c -o input_for_hello.ll

# HelloWorld pass Processing an LLVM IR file
$LLVM_DIR/bin/opt.exe -load-pass-plugin HelloWorld.dll -passes=hello-world -S input_for_hello.ll -o output_hello.ll

# 方法一
# Generate an LLVM obj file
$LLVM_DIR/bin/bin/llc.exe output_hello.ll -o output_hello.s
$LLVM_DIR/bin/bin/clang-cl.exe /c output_hello.s -o output_hello.o

# 方法二
# Generate an LLVM obj file
$LLVM_DIR/bin/clang-cl.exe -c output_hello.ll -o output_hello.o

# Generate an LLVM exe file
$LLVM_DIR/bin/clang-cl.exe output_hello.ll -o output_hello.exe

# 详情参考
$LLVM_DIR/bin/clang-cl.exe -v hello.c -o hello.exe
```

Common Problems
==========
  * JsonTest.cpp得转换成UTF8-BOM编码格式
  * clang-objc-fuzzer失败提示DummyClangFuzzer.o外部连接无法连接LLVMFuzzerInitialize符号, 在ClangObjectiveCFuzzer.cpp添加`extern "C" int LLVMFuzzerInitialize(int *argc, char ***argv) { return 0; }`
  * Hikari在llvm 10上FunctionCallObfuscate不兼容,目前没详细看,编译是过不了的.

## 有问题可以关注下我的[Github](https://github.com/Amoystyle/LLVM-VS)
