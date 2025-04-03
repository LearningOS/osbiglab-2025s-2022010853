# OS LAB Report - week7

## OS大赛赛题协作文档

编写 `LTP`模块内容，进行分工，负责编写题目描述与样例输出，提交 `Pull Request`

链接：[oskernel-testsuits-cooperation/README.md at master · oscomp/oskernel-testsuits-cooperation](https://github.com/oscomp/oskernel-testsuits-cooperation/blob/master/README.md)

## 测试补全testcase_list

将 `testcase_list`全集进行测试，尽可能添加了当前可以 `pass`的测例，同步更新到天梯仓库。

## 熟悉编译调试流程

更改希望测试的测例：修改 `undefined-os/apps/bin/testcase_list`

进行编译调试： `make LOG=info AX_TESTCASE=bin BLK=y NET=y ACCEL=n FEATURES=fp_simd,lwext4_rs ARCH=riscv64 run`

## Debug

```rust
pub unsafe fn sys_writev(fd: c_int, iov: *const ctypes::iovec, iocnt: c_int) -> ctypes::ssize_t {
    //...
    //如果是 Err(error)，立即从当前函数返回这个错误。
    let result = write_impl(fd, iov.iov_base, iov.iov_len)?;
    //如果是 Err(_)，返回默认值 0 并赋值给 result，不会提前返回。
    let result = write_impl(fd, iov.iov_base, iov.iov_len).unwrap_or(0);
    //...
}
```

`score+=4`

## 熟悉 Rust 语言

主要参考资料：[程序设计训练（Rust）](https://lab.cs.tsinghua.edu.cn/rust/)

进一步熟悉并学习 `rust`语言的语法和特性。

## For next week

- 进一步熟悉 `rust`语言特性
- 当前存在四个已实现测例在其他三个平台均可 `pass`，但是只是在 `x86_64`架构出现 `page_fault`，希望解决这个 `bug`，`score+=8`
- 独自实现一个sys_call并通过测例，`score++`
