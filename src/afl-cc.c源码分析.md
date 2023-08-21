# afl-cc.c源码分析

这个文件主要处理的逻辑是：
1、根据环境变量进行一系列的处理

+ 如果存在DEBUG环境变量，不输出详细信息，
+ 如果有对插桩文件\源码进行筛选，设置标志位，后续处理
+ 找到lto、llvm、gcc_plugin、gcc编译模式需要的so文件
  + SanitizerCoverageLTO.so
  + cmplog-routines-pass.so
  + afl-gcc-pass.so

+ 如果AFL_CC_COMPILER设置comipler_mode了，就直接可以取得值为LTO、LLVM、GCC_PLUGIN、GCC其中一个

+ 从主函数参数中判断编译器使用的是afl-clang-\*还是afl-gcc-\*等等，从而对cang_mode、compiler_mode等变量赋值，然后判断是否选择了对应可选的compiler_mode，可以从以下解释管中窥豹

  ```shell
                                         |------------- FEATURES -------------|
  MODES:                                  NCC PERSIST DICT   LAF CMPLOG SELECT
    [LLVM] LLVM:             AVAILABLE
        PCGUARD              AVAILABLE      yes yes     module yes yes    yes
        CLASSIC                    no  yes     module yes yes    yes
          - NORMAL
          - CALLER
          - CTX
          - NGRAM-{2-16}
    [LTO] LLVM LTO:          DEFAULT       
        PCGUARD              DEFAULT      yes yes     yes    yes yes    yes
        CLASSIC                           yes yes     yes    yes yes    yes
    [GCC_PLUGIN] gcc plugin: AVAILABLE
        CLASSIC              DEFAULT      no  yes     no     no  no     yes
    [GCC/CLANG] simple gcc/clang: AVAILABLE [SELECTED]
        CLASSIC              DEFAULT      no  no      no     no  no     no
  
  Modes:
    To select the compiler mode use a symlink version (e.g. afl-clang-fast), set
    the environment variable AFL_CC_COMPILER to a mode (e.g. LLVM) or use the
    command line parameter --afl-MODE (e.g. --afl-llvm). If none is selected,
    afl-cc will select the best available (LLVM -> GCC_PLUGIN -> GCC).
    The best is LTO but it often needs RANLIB and AR settings outside of afl-cc.
  
  ```

+ 判断如何对g++进行封装的一系列步骤

+ 判断插桩模式是否支持llvm

+ 通过以下函数开始拼接gcc的参数

  ```C
  //在以下这个函数对gcc进行封装
  edit_params(argc, argv, envp);
  ```



+ 以上描述的内容可以使用gdb或者以下指令观察理解

```shell
afl-gcc --verbose hello.c -o hello
```

+ afl-cc编译完后，需要调用afl-as进行汇编，将汇编语言变为机器码