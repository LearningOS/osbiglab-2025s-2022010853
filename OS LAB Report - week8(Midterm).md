# OS LAB Report - week8(Midterm)

## **OS大赛赛题协作文档**

编写 `LTP`模块内容，进行分工，负责编写题目描述与样例输出，提交 `Pull Request`，现已被合并到主分支中。

链接：[introduction for ltp by vincent-sjh · Pull Request #10 · oscomp/oskernel-testsuits-cooperation](https://github.com/oscomp/oskernel-testsuits-cooperation/pull/10)

## syscall实现

### sys_prlimit64

官方文档：[getrlimit(2) - Linux manual page](https://man7.org/linux/man-pages/man2/getrlimit.2.html)

```c
   #include <sys/resource.h>

   int getrlimit(int resource, struct rlimit *rlim);
   int setrlimit(int resource, const struct rlimit *rlim);

   int prlimit(pid_t pid, int resource,
               const struct rlimit *_Nullable new_limit,
               struct rlimit *_Nullable old_limit);

   struct rlimit {
       rlim_t  rlim_cur;  /* Soft limit */
       rlim_t  rlim_max;  /* Hard limit (ceiling for rlim_cur) */
   };

   typedef /* ... */  rlim_t;  /* Unsigned integer type */
```

`sys_prlimit64` 系统调用是允许用户或程序获取当前进程或指定进程的资源限制信息，或者设置新的资源限制。相比于早期的类似调用（如 `getrlimit` 和 `setrlimit`），`prlimit64 `提供了对 64 位系统更好的支持，能够处理更大的资源数值，尤其是在管理内存、文件描述符数量、CPU 时间等资源时更为精确，可以通过 `prlimit64`实现更细粒度的资源控制，确保系统稳定性或限制特定进程的资源占用。它接受一个进程 `PID`作为参数，可以操作当前进程或目标进程的资源限制，常见的资源包括最大文件大小、堆栈大小、打开文件描述符数量等。

```rust
// src/syscall.rs
// 在handle_syscall函数中进行注册
    Sysno::prlimit64 => sys_prlimit64(
        tf.arg0() as _,
        tf.arg1() as _,
        tf.arg2().into(),
        tf.arg3().into(),
    )
//core/src/task.rs
//添加所需结构体Rlimit的定义
#[derive(Debug, Clone, Copy)]
#[repr(C)]
pub struct Rlimit{
    pub rlim_cur: u32 ,
    pub rlim_max: u32,
}

impl Default for Rlimit {
    fn default() -> Self {
        Rlimit {
            //自定义初始值最开始设置为0，导致bug
            //原因是这个结构体的意义是进程的资源限制，初始值为0会导致进程无法使用资源，于是设置了一个较大的初始值
            rlim_cur: 65535, // 自定义初始值
            rlim_max: 65535, // 自定义初始值
        }
    }
}

// TaskExt 结构体增添对应资源的成员变量，类型为Rlimit
/// Task extended data for the monolithic kernel.
pub struct TaskExt {
    ///...
    // RLIMIT_AS：进程的最大虚拟内存大小（字节）。
    pub rlimit_as: Cell<Rlimit>,
    // RLIMIT_CORE：核心转储文件（core dump）的最大大小。
    pub rlimit_asc: Cell<Rlimit>,
    // RLIMIT_CPU：CPU 时间限制（秒）。
    pub rlimit_cpu: Cell<Rlimit>,
    // RLIMIT_DATA：数据段的最大大小。
    pub rlimit_data: Cell<Rlimit>,
    // RLIMIT_FSIZE：创建文件的最大大小。
    pub rlimit_fsize: Cell<Rlimit>,
    // RLIMIT_NOFILE：打开文件描述符的最大数量。
    pub rlimit_nofile: Cell<Rlimit>,
    // RLIMIT_STACK：栈的最大大小。
    pub rlimit_stack: Cell<Rlimit>,
}

impl TaskExt {
    pub fn new(
        proc_id: usize,
        uctx: UspaceContext,
        aspace: Arc<Mutex<AddrSpace>>,
        heap_bottom: u64,
    ) -> Self {
        Self {
			//...
            rlimit_as: Cell::new(Rlimit::default()),
            rlimit_asc: Cell::new(Rlimit::default()),
            rlimit_cpu: Cell::new(Rlimit::default()),
            rlimit_data: Cell::new(Rlimit::default()),
            rlimit_fsize: Cell::new(Rlimit::default()),
            rlimit_stack: Cell::new(Rlimit::default()),
            rlimit_nofile: Cell::new(Rlimit::default()),
        }
    }
    //配套函数，用于获取和设置成员变量的值
    pub fn set_rlimit_nofile(&self, new_value:  Rlimit) {
        self.rlimit_nofile.set(new_value);
    }
    pub fn get_rlimit_nofile(&self) -> Rlimit {
        self.rlimit_nofile.get()
    }
    //...
//api/src/imp/task/thread.rs
//sys_prlimit64主函数体
    pub fn sys_prlimit64(
    _pid: i32,
    resource: u32,
    new_limit: UserConstPtr<Rlimit>,
    old_limit: UserPtr<Rlimit>,
) -> LinuxResult<isize> {
    let curr = current();
    let task = curr.task_ext();
    match resource {
        RLIMIT_NOFILE => {
            //设置新的资源限制
            let old_num: Rlimit = task.get_rlimit_nofile();
            let new_limit = new_limit.nullable(UserConstPtr::get)?;
            if let Some(new_limit) = new_limit {
                unsafe { task.set_rlimit_nofile(*new_limit); }
            }
			//获取旧的资源限制
            let old_limit = old_limit.nullable(UserPtr::get)?;
            if let Some(old_limit) = old_limit {
                unsafe {
                    *old_limit = old_num;
                }
            }
        },
        //...
        _ => {

        }
    }
    Ok(0)
}
	//api/src/imp/fs/fd_ops.rs
    //添加资源限制相关支持
    pub fn sys_dup(old_fd: c_int) -> LinuxResult<isize> {
    let curr = current();
    let task = curr.task_ext();
    //如果发现打开的文件数超过资源限制就报错
    //EMFILE = 24,Too many open files
    if FD_TABLE.read().count()  >= task.get_rlimit_nofile().rlim_cur as usize{
       return Err(LinuxError::EMFILE);
    }
    Ok(api::sys_dup(old_fd) as _)
	}

```

### sys_rt_sigprocmask

官方文档：[sigprocmask(2) - Linux manual page](https://man7.org/linux/man-pages/man2/sigprocmask.2.html)

```c
       #include <signal.h>

       /* Prototype for the glibc wrapper function */
       int sigprocmask(int how, const sigset_t *_Nullable restrict set,
                                  sigset_t *_Nullable restrict oldset);

       #include <signal.h>           /* Definition of SIG_* constants */
       #include <sys/syscall.h>      /* Definition of SYS_* constants */
       #include <unistd.h>

       /* Prototype for the underlying system call */
       int syscall(SYS_rt_sigprocmask, int how,
                                  const kernel_sigset_t *_Nullable set,
                                  kernel_sigset_t *_Nullable oldset,
                                  size_t sigsetsize);

       /* Prototype for the legacy system call */
       [[deprecated]] int syscall(SYS_sigprocmask, int how,
                                  const old_kernel_sigset_t *_Nullable set,
                                  old_kernel_sigset_t *_Nullable oldset);
```

`sys_rt_sigprocmask` 系统调用用于管理进程的信号掩码，控制哪些信号被进程暂时阻塞或允许接收，通过检查当前信号掩码、将指定信号添加到掩码中以阻塞、从掩码中移除信号以解除阻塞，或用新信号集合替换现有掩码来实现功能，对应 `SIG_BLOCK`、`SIG_UNBLOCK 和 SIG_SETMASK `三种操作选项来调整信号状态，常用于需要精确控制信号处理时机的场景，比如在关键代码执行时暂时屏蔽 `SIGINT` 或 `SIGTERM `以避免中断，随后在安全时机恢复信号处理，为信号管理提供了灵活而强大的支持。

```rust
// src/syscall.rs
// 在handle_syscall函数中进行注册
    Sysno::rt_sigprocmask => sys_rt_sigprocmask(
        tf.arg0() as _,
        tf.arg1().into(),
        tf.arg2().into(),
        tf.arg3() as _,
    )
//core/src/task.rs
//添加所需结构体SigSet的定义
#[derive(Debug, Clone, Copy)]
#[repr(C)]
pub struct SigSet {
    pub bits: [usize; 2],
}
impl SigSet {
    
    pub fn add_from(&mut self, other: *const SigSet) {
        unsafe{
            self.bits[0] |= (*other).bits[0];
            self.bits[1] |= (*other).bits[1];
        }
    }
    pub fn remove_from(&mut self, other: *const SigSet) {
        unsafe{
            self.bits[0] &= !(*other).bits[0];
            self.bits[1] &= !(*other).bits[1];
        }
    }
}

// TaskExt 结构体增添对应资源的成员变量，类型为SigSet
/// Task extended data for the monolithic kernel.
pub struct TaskExt {
    ///...
    // signal mask
    pub signal_mask: Cell<SigSet>
}

impl TaskExt {
    pub fn new(
        proc_id: usize,
        uctx: UspaceContext,
        aspace: Arc<Mutex<AddrSpace>>,
        heap_bottom: u64,
    ) -> Self {
        Self {
			//...
            signal_mask: Cell::new(SigSet {
                bits: [0, 0],
            })
        }
    }
    //配套函数，用于获取和设置成员变量的值
    pub fn add_signal(&self, other: *const SigSet) {
        let mut prev_signal_mask = self.signal_mask.get();
        prev_signal_mask.add_from(other);
        self.signal_mask.set(prev_signal_mask);
    }
    pub fn remove_signal(&self, other: *const SigSet) {
        let mut prev_signal_mask = self.signal_mask.get();
        prev_signal_mask.remove_from(other);
        self.signal_mask.set(prev_signal_mask);
    }
    pub fn get_signal_mask(&self) -> SigSet {
        self.signal_mask.get()
    }
    pub fn set_signal_mask(&self, mask: *const SigSet) {
        unsafe {self.signal_mask.set(*mask);}
    }
 
 //api/src/imp/signal.rs
 //sys_rt_sigprocmask主函数体
 pub fn sys_rt_sigprocmask(
    how: i32,
    set: UserConstPtr<SigSet>,
    oldset: UserPtr<SigSet>,
    _sigsetsize: usize,
) -> LinuxResult<isize> {
    let curr = current();
    let taskext = curr.task_ext();
    let oldset = oldset.nullable(UserPtr::get)?;
    if let Some(oldset) = oldset {
        unsafe {
            *oldset = taskext.get_signal_mask();
        }
    }
    let set = set.nullable(UserConstPtr::get)?;
    if let Some(set) = set {
        match how {
            // SIG_BLOCK = 0
            0 => taskext.add_signal(set),
            // SIG_UNBLOCK=1
            1 => taskext.remove_signal(set),
            // SIG_SETMASK
            2 => taskext.set_signal_mask(set),
            _ => {}
        }
    }
    Ok(0)
}
```

### sys_mkdir

官方文档：[mkdir(2) - Linux manual page](https://man7.org/linux/man-pages/man2/mkdir.2.html)

```c
#include <sys/stat.h>

int mkdir(const char *pathname, mode_t mode);

#include <fcntl.h>           /* Definition of AT_* constants */
#include <sys/stat.h>

int mkdirat(int dirfd, const char *pathname, mode_t mode);
```

`sys_mkdir` 系统调用用于在指定路径下创建一个新的目录，通过提供目录路径和权限模式作为参数，快速创建一个符合需求的目录。

```rust
// src/syscall.rs
// 在handle_syscall函数中进行注册
#[cfg(target_arch = "x86_64")]
Sysno::mkdir => sys_mkdir(tf.arg0().into(), tf.arg1() as _)

//api/src/imp/task/thread.rs
//sys_mkdir主函数体
pub fn sys_mkdir(path: UserConstPtr<c_char>, mode: u32) -> LinuxResult<isize> {
    //sys_mkdirat(AT_FDCWD as i32, path, mode);
    let path = path.get_as_str()?;
    if mode != 0 {
        info!("directory mode not supported.");
    }
    axfs::api::create_dir(path).map(|_| 0).map_err(|err| {
        warn!("Failed to create directory {path}: {err:?}");
        err.into()
    })
}
```

### sys_statx

官方文档：[statx(2) - Linux manual page](https://man7.org/linux/man-pages/man2/statx.2.html)

在重写了支持多个架构的`stat`系列函数之后，发现几个调用了stat系列`syscall`的测例除了`loongarch64`的三个架构都可以通过，仅有`loongarch64`架构出现fail，后调试研究发现仅有`loongarch`系列调用的`syscall`为`statx`，而`statx`我们并未进行重构，所以成功锁定问题在于`sys_statx`系统调用出现问题。

```rust
//api/src/imp/fs.rs
// sys_statx原函数，克隆原始仓库时即存在，并非我们团队所编写
pub fn sys_statx(
    dirfd: i32,
    pathname: UserConstPtr<c_char>,
    flags: u32,
    _mask: u32,
    statxbuf: UserPtr<StatX>,
) -> LinuxResult<isize> {

    let path = pathname.get_as_str()?;

    const AT_EMPTY_PATH: u32 = 0x1000;
    if path.is_empty() {
        if flags & AT_EMPTY_PATH == 0 {
            return Err(LinuxError::EINVAL);
        }
        // Alloc a new space for stat struct
		//...
        Ok(0)
    } else {
        //问题在这里，原有实现不完整，在路径非空的时候直接返回ENOSYS，我们对其进行了重写。
        Err(LinuxError::ENOSYS)
    }
}
```

```rust
//进行重写
pub fn sys_statx(
    dir_fd: c_int,
    path: UserConstPtr<c_char>,
    flags: c_int,
    _mask: c_uint,
    statx_buf: UserPtr<StatX>,
) -> LinuxResult<isize> {
    // get params
    let path = path.get_as_str().unwrap_or("");
    let statx_buf = statx_buf.get()?;

    // perform syscall
    let result = (|| -> LinuxResult<_> {
        //当前目录
        if dir_fd < 0 && dir_fd != AT_FDCWD as i32 {
            return Err(LinuxError::EBADFD);
        }
        
        //空路径权限
        if path.is_empty() && (flags & AT_EMPTY_PATH == 0) {
            return Err(LinuxError::ENOENT);
        }
        // TODO: some flags are ignored
        let follow_symlinks = flags & AT_SYMLINK_NOFOLLOW == 0;
        let file_status = sys_stat_impl(dir_fd, path, follow_symlinks)?;
        // TODO: add more fields
        Ok(StatX::from(file_status))
    })();

    // check result
    match result {
        Ok(statx) => {
            debug!(
                "[syscall] statx(dirfd={}, pathname={:?}, flags={}, mask={}, statx_buf={:?}) = {}",
                dir_fd, path, flags, _mask, statx, 0
            );
            // copy to user space
            unsafe { statx_buf.write(statx.into()) }
            Ok(0)
        }
        Err(err) => {
            debug!(
                "[syscall] statx(dirfd={}, pathname={:?}, flags={}, mask={}, statx_buf={:p}) = {:?}",
                dir_fd, path, flags, _mask, statx_buf, err
            );
            Err(err)
        }
    }
}
```

以及StatX结构体定义经过比对发现缺少一块大小为16的`pad`会导致解析错误，进行了填充修改。

官方文档：[statx(2) - Linux manual page](https://man7.org/linux/man-pages/man2/statx.2.html)中缺少了这个`pad`的定义，之后的工作中要以C库的具体定义为准。

```rust
#[repr(C)]
#[derive(Debug, Default)]
pub struct StatX {
	//File mode (permissions).
    pub stx_mode: u16,
    /// padding(缺少)
    pub _pad0: u16,
    /// Inode number.
    pub stx_ino: u64,
   //...
}
// stat.h
struct statx {
	/* 0x00 */
	__u32	stx_mask;	/* What results were written [uncond] */
	__u32	stx_blksize;	/* Preferred general I/O size [uncond] */
	__u64	stx_attributes;	/* Flags conveying information about the file [uncond] */
	/* 0x10 */
	__u32	stx_nlink;	/* Number of hard links */
	__u32	stx_uid;	/* User ID of owner */
	__u32	stx_gid;	/* Group ID of owner */
	__u16	stx_mode;	/* File mode */
	__u16	__spare0[1];
	/* 0x20 */
	__u64	stx_ino;	/* Inode number */
	__u64	stx_size;	/* File size */
	__u64	stx_blocks;	/* Number of 512-byte blocks allocated */
	__u64	stx_attributes_mask; /* Mask to show what's supported in stx_attributes */
	/* 0x40 */
	struct statx_timestamp	stx_atime;	/* Last access time */
	struct statx_timestamp	stx_btime;	/* File creation time */
	struct statx_timestamp	stx_ctime;	/* Last attribute change time */
	struct statx_timestamp	stx_mtime;	/* Last data modification time */
	/* 0x80 */
	__u32	stx_rdev_major;	/* Device ID of special file [if bdev/cdev] */
	__u32	stx_rdev_minor;
	__u32	stx_dev_major;	/* ID of device containing file [uncond] */
	__u32	stx_dev_minor;
	/* 0x90 */
	__u64	stx_mnt_id;
	__u64	__spare2;
	/* 0xa0 */
	__u64	__spare3[12];	/* Spare space for future expansion */
	/* 0x100 */
};
```



## Issue(改进建议)

### 无效测例

在调试`glibc`的测例`fpclassify_invalid_ld80`过程中，发现本地`wsl`运行此测例行为和输出结果与在实验框架下相同，但流水线测评为`fail`，拿不到分数。


经过沟通和确认发现`fpclassify_invalid_ld80`的测例代码在`x86_64`下编译工具链会出现问题，后联系了编写测评流水线的同学进行修改，相关负责同学及时进行修改，现在这个测例会自动判对，问题解决。

### 错误测例

发现在`stat`系列系统调用仅仅支持x86_64架构的时候`basic`相关测例会`success`拿到分数，但是在重写了`stat`系列函数，添加了对于其他三个架构的支持之后`basic`里面的相关测例`fail`，最后发现`basic`相关测例仅仅兼容`x86_64`，联系了负责测例的同学进行修改，相关负责同学及时进行修改，问题解决。

### 测评脚本逻辑问题

在调试`glibc`的测例`test_getdents`的过程中，发现正常输出了正确结果但是流水线测评为`fail`，拿不到分数。

```
========== START test_getdents ==========
open fd:3
getdents fd:499
getdents success.
.

========== END test_getdents ==========
```

之后阅读评测脚本发现脚本会判定输出的目录名长度大于1，但是目录名有可能为“.”，长度恰好为1导致判错，联系了编写测评流水线的同学进行修改，相关负责同学及时进行修改，判断改为大于等1，问题解决。


在评测脚本中如果出现相同的行输出，会导致评测脚本正确输出统计量翻倍（4->8），导致错误，联系了编写测评流水线的同学进行修改，相关负责同学及时进行修改，问题解决。


## For next week

- 维持榜一，得分升高到`570~580`左右，主要关注`thread`相关测例。
- 完善并整理已实现的系统调用（修改位置&代码规范）

