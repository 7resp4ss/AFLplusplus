# afl-as.c源码分析

+ as：**汇编器**，把汇编语言代码转换为机器码

这部分源码比较简单，主要逻辑就以下三个函数

```c
edit_params(argc, argv);	//对as程序进行封装

if (!just_version) { add_instrumentation(); }	//白盒插桩的主要在这
  if (!(pid = fork())) {
   	//子进程的逻辑
    execvp(as_params[0], (char **)as_params);
    FATAL("Oops, failed to execute '%s' - check your PATH", as_params[0]);
  }
//等待子进程执行完毕
if (waitpid(pid, &status, 0) <= 0) { PFATAL("waitpid() failed"); }
```

+ 为什么要用子进程进行汇编？

  这其实是因为我们的execvp执行的时候，会用`as_params[0]`来完全替换掉当前进程空间中的程序，如果不通过子进程来执行实际的as，那么后续就无法在执行完实际的as之后，还能unlink掉modified_file

## add_instrumentation实现

主要逻辑：处理输入文件，生成modified_file，将instrumentation插入所有适当的位置。

+ 打开两个文件，一个是inf（输入文件：input_file）、一个是outfd（输出文件：modified_file）

+ 逐行读入文件，对于读入的每一行，先判断插桩的条件是否满足（`!pass_thru && !skip_intel && !skip_app && !skip_csect && instr_ok && instrument_next`），如果满足则直接插入插桩代码（`trampoline_fmt_64` 或`trampoline_fmt_32`，根据是`32`位还是`64`位），插桩完成后，表明该基本块已经完成插桩，后面的代码无需插桩，将`instrument_next`置位`0`

  ```C
  while (fgets(line, MAX_LINE, inf)) {
  ...
  }
  ```

  注意到通过`while`循环对汇编进行不同程度的插桩！

+ 无论是否插桩，都将当行代码写入到`modified_file`中

+ `pass_thru`标志位是在`edit_params`中设定的；`skip_next_label`标志位是为了处理`OpenBSD`系统上的跳转表而设置的标志位；`\t.text`、`\t.section\t.text`等开头的行则说明接下来是`text`段，可能需要进行插桩因此要设置`instr_ok`标志位，再次遇到`\t.section`或`\t.bss`等说明到了其它的段，需要将`instr_ok`标志位置`0`。`skip_csect`则是用来标志`off-flavor assembly`；`skip_intel`用来处理`intel`汇编语法，`afl`只对`AT&T`汇编表示进行插桩；`skip_app`用来标志`ad-hoc __asm__`（不太明白这是啥）。

+ 最需要关注的是`instr_ok`标志位，该标志位用来表示是否处于`text`段，如果处于则可能需要进行插桩，否则无需进行插桩。其它标志位正常情况下在`ubuntu`系统下`gcc`生成的汇编代码，应该不会有对应的代码出现。

+ 对于在`text`段，需要插桩的代码，注释中有比较良好的说明，如下所示。

  主要是需要在各个基本块的入口进行插桩，具体来说：

  + 对于`main`函数的入口（`^main:`）需要插桩，因为需要初始化；对于条件条件的标签后面（`gcc`是`^.L0:`，`clang`是`^.LBB0_0:`）需要插桩，因为它是条件跳转的目标地址；对于跳转指令（`^\tjnz foo`）后面也需要插桩，因为该指令的后面形成了分支。

  + 而对于注释（`^# BB#0:`以及`^ # BB#0:`）不需要插桩；绝对跳转的目标地址（`^.Ltmp0:`、`^.LC0`以及`^.LBB0_0:`）不需要插桩，因为没有形成新的分支或路径；绝对跳转指令（`^\tjmp foo`）也无需插桩。

  + 对于条件跳转指令，需要在条件跳转指令后面以及在跳转指令的目标标签后面都需要插桩，因为在条件跳转指令后形成了两条分支，需要对其插桩监控以查看是否执行了更多的路径。

    ```c
    //条件跳转指令
    if (line[0] == '\t') {
    
          if (line[1] == 'j' && line[2] != 'm' && R(100) < inst_ratio) {
    
            fprintf(outf, use_64bit ? trampoline_fmt_64 : trampoline_fmt_32,
                    R(MAP_SIZE));//通过这里进行插桩标识
    
            ins_lines++;
    
          }
    
          continue;
    
        }
    
    //条件跳转的标签
            /* Label of some sort. This may be a branch destination, but we need to
           tread carefully and account for several different formatting
           conventions. */
    
    #ifdef __APPLE__
    
        /* Apple: L<whatever><digit>: */
    
        if ((colon_pos = strstr(line, ":"))) {
    
          if (line[0] == 'L' && isdigit(*(colon_pos - 1))) {
    
    #else
    
        /* Everybody else: .L<whatever>: */
    
        if (strstr(line, ":")) {
    
          if (line[0] == '.') {
    
    #endif /* __APPLE__ */
    
            /* .L0: or LBB0_0: style jump destination */
    
    #ifdef __APPLE__
    
            /* Apple: L<num> / LBB<num> */
    
            if ((isdigit(line[1]) || (clang_mode && !strncmp(line, "LBB", 3)))
                && R(100) < inst_ratio) {
    
    #else
    
            /* Apple: .L<num> / .LBB<num> */
    
            if ((isdigit(line[2]) || (clang_mode && !strncmp(line + 1, "LBB", 3)))
                && R(100) < inst_ratio) {
    
    #endif /* __APPLE__ */
    
              /* An optimization is possible here by adding the code only if the
                 label is mentioned in the code in contexts other than call / jmp.
                 That said, this complicates the code by requiring two-pass
                 processing (messy with stdin), and results in a speed gain
                 typically under 10%, because compilers are generally pretty good
                 about not generating spurious intra-function jumps.
    
                 We use deferred output chiefly to avoid disrupting
                 .Lfunc_begin0-style exception handling calculations (a problem on
                 MacOS X). */
    
              if (!skip_next_label) instrument_next = 1; else skip_next_label = 0;
    
            }
    
          } else {
    
            /* Function label (always instrumented, deferred mode). */
    
            instrument_next = 1;
    
          }
    
        }
    
      }
    ```

  + 在整个汇编代码遍历完成后，如果无需插桩的话，则不需要加入额外的`main_payload`汇编代码；如果经历过插桩的话，则加入`main_payload`，`main_payload`的作用是插桩代码的主体功能的实现。

    最终关闭相应的文件句柄，并输出信息，完成`.s`文件的插桩。

  ```c
   // afl-as.c: 454
    if (ins_lines)
      fputs(use_64bit ? main_payload_64 : main_payload_32, outf);	//在这里进行插桩
  
    if (input_file) fclose(inf);
    fclose(outf);
  
    if (!be_quiet) {
  
      if (!ins_lines) WARNF("No instrumentation targets found%s.",
                            pass_thru ? " (pass-thru mode)" : "");
      else OKF("Instrumented %u locations (%s-bit, %s mode, ratio %u%%).",
               ins_lines, use_64bit ? "64" : "32",
               getenv("AFL_HARDEN") ? "hardened" : 
               (sanitizer ? "ASAN/MSAN" : "non-hardened"),
               inst_ratio);
  
    }
  ```

  

  + 可以从以下理解：

  ```C
      /* If we're in the right mood for instrumenting, check for function
         names or conditional labels. This is a bit messy, but in essence,
         we want to catch:
  
           ^main:      - function entry point (always instrumented)
           ^.L0:       - GCC branch label
           ^.LBB0_0:   - clang branch label (but only in clang mode)
           ^\tjnz foo  - conditional branches
  
         ...but not:
  
           ^# BB#0:    - clang comments
           ^ # BB#0:   - ditto
           ^.Ltmp0:    - clang non-branch labels
           ^.LC0       - GCC non-branch labels
           ^.LBB0_0:   - ditto (when in GCC mode)
           ^\tjmp foo  - non-conditional jumps
  
         Additionally, clang and GCC on MacOS X follow a different convention
         with no leading dots on labels, hence the weird maze of #ifdefs
         later on.
  
       */
  ```

### main_payload

这部分需要分析



### 总结

+ instr_ok代表了输入的汇编在.text部分

## afl-fast-clang插桩简要分析

主要文件在[AFLplusplus](https://github.com/7resp4ss/AFLplusplus/tree/stable)/[instrumentation](https://github.com/7resp4ss/AFLplusplus/tree/stable/instrumentation)里（应该）

不懂llvm🤔，以后填坑



## 参考链接

+ [fuzzer AFL 源码分析（一）- 编译 - 跳跳糖 (tttang.com)](https://tttang.com/archive/1595/)
+ [sakuraのAFL源码全注释（一）-安全客 - 安全资讯平台 (anquanke.com)](https://www.anquanke.com/post/id/213430#h2-4)