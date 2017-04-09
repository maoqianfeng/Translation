# Poor Man's Sampling Profiler

官网英文原文地址：https://dev.px4.io/advanced-profiling.html

本节介绍如何通过分析来评估PX4系统的性能。

Credits for the idea belong to
[Mark Callaghan and Domas Mituzas](https://dom.as/2009/02/15/poor-mans-contention-profiling/).

这个棒棒的主意是Mark Callaghan 和 Domas Mituzas的 。

## 方法



PMSP是一个shell脚本，它会定期中断固件的运行，并对当前堆栈踪迹进行采样。采样的堆栈踪迹会被追加到一个文本文件中。一旦采样结束（通常要花一个小时，或者更长时间），收集的堆栈踪迹会被 *folded*。*folding*的结果是另一个含有相同堆栈踪迹的文本文件，相同的堆栈踪迹（比如那些在程序中相同点获得的）会合并到一起，并记录发生的次数。然后这个折叠的堆栈信息会被送入可视化脚本，这个脚本我们使用[FlameGraph - an open source stack trace visualizer](http://www.brendangregg.com/flamegraphs.html).

## 执行


脚本的位置在 `Debug/poor-mans-profiler.sh`。一旦开始运行，脚本会以指定的时间间隔运行指定数量的采样。收集的采样信息会存储在位于系统的临时文件夹(例如 `/tmp`)下的文本文件中。一旦采样结束，脚本会自动调用这个堆栈文件夹，输出信息会存储在临时文件夹下的相邻文件中。如果堆栈`folded`成功，脚本会调用 FlameGraph 脚本，并把结果保存在交互式的SVG文件中。请注意并不是所有的图片浏览器都支持交互式图片；推荐在网页浏览器中打开SVG结果。

FlameGraph脚本必须在 `PATH` 中，否则 PMSP 拒绝执行。

PMSP使用GDB收集堆栈踪迹。目前使用的是 `arm-none-eabi-gdb` ,未来可能会加入其他工具链。

为了能够将内存位置映射到符号上，脚本需要被引用到当前在目标上运行的可执行文件上。这通过选项 `--elf=<file>` 来实现。这个参数是一个指向当前执行的 ELF 的路径（相对于代码库的根路径）。  

用法示例:

```bash
./poor-mans-profiler.sh --elf=build_px4fmu-v4_default/src/firmware/nuttx/firmware_nuttx --nsamples=30000
```

注意每次运行脚本都会覆盖旧的堆栈。如果你想向旧的堆栈追加而不是重写，使用选项 `--append` ：

```bash
./poor-mans-profiler.sh --elf=build_px4fmu-v4_default/src/firmware/nuttx/firmware_nuttx --nsamples=30000 --append
```

正如人们怀疑的那样， `--append` 加上 `--nsamples=0` 会指示脚本在不访问目标的情况下重新生成SVG。


请阅读脚本深入理解脚本是如何工作的。

## 理解输出

下面是一个样例输出的截屏（并不是交互性的）

![FlameGraph Example](flamegraph-example.png)


在帧图上，水平方向表示堆栈帧，每个帧的宽度正比于它被采样的次数。反过来，函数最终被采样的次数正比于执行的持续时间频率。

## 可能的问题

The script was developed as an ad-hoc solution, so it has some issues.
Please watch out for them while using it:

* If GDB is malfunctioning, the script may fail to detect that, and continue running.
  In this case, obviously, no usable stacks will be produced.
  In order to avoid that, the user should periodically check the file `/tmp/pmpn-gdberr.log`,
  which contains the stderr output of the most recent invocation of GDB.
  In the future the script should be modified to invoke GDB in quiet mode, where it will indicate
  issues via its exit code.

* Sometimes GDB just stucks forever while sampling the stack trace.
  During this failure, the target will be halted indefinitely.
  The solution is to manually abort the script and re-launch it again with the `--append` option.
  In the future the script should be modified to enforce a timeout for every GDB invocation.

* Multithreaded environments are not supported.
  This does not affect single core embedded targets, since they always execute in one thread,
  but this limitation makes the profiler incompatible with many other applications.
  In the future the stack folder should be modified to support multiple stack traces per sample.
