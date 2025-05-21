# OS LAB Report - week13

## `aarch64`架构兼容

整合当前的所有实现推送测试后发现`aarch64`架构无法兼容，`sys_mprotect`显示内存块未分配(`Unmapped`)，无法设置`flag`。

```rust
aspace.protect(start_addr, length, permission_flags.into())?;
```

于是我在`sys_mprotect`中调用`aspace`的`check_region_access`方法（但是给予的检验参数为`0`，理论上`region`如果被`map`过就应当返回`true`，否则`false`），用来验证内存块是否真的被`map`过。

```rust
let zero_flag = MappingFlags::READ - MappingFlags::READ;
aspace.check_region_access(region,zero_flag);
```

验证结果是内存块被`map`过，但底层`page_table`内并未支持`aarch64`架构实现，后更新了`page_table_multiarch`的版本后问题解决。

## `socket`相关`syscall`实现

参照BattiestStone4同学实现`socket`相关支持。

```rust
		Sysno::setsockopt => sys_setsockopt(),
        Sysno::sendto => sys_sendto(),
        Sysno::recvfrom => sys_recvfrom(),
        Sysno::shutdown => sys_shutdown(),
        Sysno::listen => sys_listen(),
        Sysno::accept => sys_accept(),
        Sysno::connect => sys_connect(),
```

实现过程中，在BattiestStone4的热心帮助下，发现最终问题在于`sys_fcntl`缺乏在`F_GETFD`和`F_GETFL`这两种参数下的支持，添加后最终通过`libc`的`socket`测例。

- `F_GETFD`：获取文件描述符的标志（如 `FD_CLOEXEC`），通常用于检查文件描述符的状态。
- `F_GETFL`：获取文件状态标志（如 `O_RDONLY`、`O_WRONLY`、`O_RDWR`、`O_NONBLOCK` 等），用来检查文件的访问模式和状态标志。

到现在为止，除了`iozone`部分的分数，仅剩下libctest的三个测例尚未通过。

### libc - left 3 undone

| case\arch               | static/dynamic | riscv64 | loongarch64 | aarch64 | x86_64 | details               |
| ----------------------- | -------------- | ------- | ----------- | ------- | ------ | --------------------- |
| `pthread_cancel_points` | ✅✅             | ❌       | ❌           | ❌       | ❌      | ——                    |
| `pthread_robust_detach` | ✅✅             | ❌       | ❌           | ❌       | ❌      | 共享内存`shm`相关支持 |
| `utime`                 | ✅✅             | ❌       | ❌           | ❌       | ❌      | `sys_utimensat`       |


## For next week

- 通过`libc`的`utime`测例—`sys_utimensat`

- 推进iozone测例—`shm`系列`syscall`实现

  - `sys_shmget`
  - `sys_shmat`
  - `sys_shmctl`

  
