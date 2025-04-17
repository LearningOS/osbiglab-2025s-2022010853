# OS LAB Report - week9

## syscall实现

针对最后两个`busybox`测例点，调用axfs的api接口实现了以下三个`syscall`。

- `sys_rmdir`
- `sys_rename`
- `sys_renameat2`

至此已经成功通过了`busybox`所有测例点。

## 测例问题统计

统计并整理了尚未通过的测例的原因，以便后续实现。

飞书链接：[‌⁤‍⁣⁡⁤‬‌⁣﻿⁡‌⁢⁣‌⁡⁣⁡﻿⁣‬⁢⁣‌‌⁡‌‍⁤⁡‌⁡‬‬﻿⁤⁢⁤⁤‬⁡⁤Undefined-OS 测例问题统计 - 飞书云文档](https://jkmxhqlj8z.feishu.cn/docx/RylfdB5PqoyxS3xXf3oconuXnhe)

| case\arch                 | static/dynamic | riscv64 | loongarch64 | aarch64 | x86_64 | details                                                 |
| ------------------------- | -------------- | ------- | ----------- | ------- | ------ | ------------------------------------------------------- |
| pthread_cancel_points     | ✅✅             | ❌       | ❌           | ❌       | ❌      | Thread 支持                                             |
| pthread_cancel            | ✅✅             | ❌       | ❌           | ❌       | ❌      | Thread 支持                                             |
| pthread_cond              | ✅✅             | ❌       | ❌           | ❌       | ❌      | Thread 支持                                             |
| pthread_tsd               | ✅✅             | ❌       | ❌           | ❌       | ❌      | Thread 支持                                             |
| pthread_robust_detach     | ✅✅             | ❌       | ❌           | ❌       | ❌      | Thread 支持                                             |
| pthread_cancel_sem_wait   | ✅❌             | ❌       | ❌           | ❌       | ❌      | Thread 支持                                             |
| pthread_cond_smasher      | ✅✅             | ❌       | ❌           | ❌       | ❌      | Thread 支持                                             |
| pthread_condattr_setclock | ✅✅             | ❌       | ❌           | ❌       | ❌      | Thread 支持                                             |
| pthread_exit_cancel       | ✅✅             | ❌       | ❌           | ❌       | ❌      | Thread 支持                                             |
| pthread_once_deadlock     | ✅✅             | ❌       | ❌           | ❌       | ❌      | Thread 支持                                             |
| pthread_rwlock_ebusy      | ✅✅             | ❌       | ❌           | ❌       | ❌      | Thread 支持                                             |
| utime                     | ✅✅             | ❌       | ❌           | ❌       | ❌      | sys_utimensat                                           |
| daemon_failure            | ✅✅             | ❌       | ❌           | ❌       | ❌      | sys_prlimit64 RLIMIT_STACK                              |
| socket                    | ✅✅             | ❌       | ❌           | ❌       | ❌      | sys_socket                                              |
| stat                      | ✅✅             | ❌       | ❌           | ❌       | ❌      | sys_unlink 在还存在文件描述符指向文件的时候直接删除文件 |
| dlopen                    | ❌✅             | ❌       | ❌           | ❌       | ❌      | ###                                                     |
| sem_init                  | ❌✅             | ❌       | ❌           | ❌       | ❌      | sys_futex                                               |
| tls_local_exec            | ❌✅             | ❌       | ❌           | ❌       | ❌      | sys_futex                                               |
| tls_init                  | ❌✅             | ❌       | ❌           | ❌       | ❌      | sys_futex                                               |
| tls_get_new_dtv           | ❌✅             | ❌       | ❌           | ❌       | ❌      | sys_futex                                               |

## For next week

- 计划阅读和参考一些OS范例的实现，主要关注thread相关syscall的支持。
- 星绽OS[asterinas/README_CN.md at main · asterinas/asterinas](https://github.com/asterinas/asterinas/blob/main/README_CN.md)
- ByteOS https://github.com/LearningOS/oscomp-test-yfblock/ 
- 通过thread相关测例。
