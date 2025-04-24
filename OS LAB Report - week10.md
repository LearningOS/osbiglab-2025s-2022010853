# OS LAB Report - week10

## syscall实现

更改测试镜像之后，一些`busybox`测例点出现问题，针对重新重新出现的问题，实现并调试通过了如下的`syscall`:

- `sys_rmdir`
- `sys_rename`
- `sys_renameat`
- `sys_renameat2`

至此已经成功通过了`busybox`所有测例点。


## 尝试合并上游Thread相关代码

- 进行了大量繁琐的调试工作，但是和原来编写的代码有很多不兼容之处，目前组内倾向于放弃直接合并，转而在我们自己代码的基础上逐渐添加和调试。
- 计划把组间交流的程度定位在阅读代码参考思路的层面，后续不再试图直接合并整个模块的代码，以免时间的浪费，也避免由于后续调试过程中由于对外来代码的不熟悉导致的调试成本。



## For next week

- 确定自己完成`Thread`相关`syscall`
- 计划阅读和参考一些OS范例的实现，主要关注thread相关syscall的支持。
- 星绽OS[asterinas/README_CN.md at main · asterinas/asterinas](https://github.com/asterinas/asterinas/blob/main/README_CN.md)
- ByteOS https://github.com/LearningOS/oscomp-test-yfblock/ 
- 通过thread相关测例。
