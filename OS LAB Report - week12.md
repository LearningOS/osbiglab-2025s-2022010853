# OS LAB Report - week12

## 兼容process/thread架构重写后的实现

重写了process/thread之后，将调用接口全部替换为新写法，一些`busybox`测例点出现问题，针对重新重新出现的问题进行调试debug，保证`busybox`所有测例点依然正确。


## 保证iozone读写正确性

原有框架下，iozone测试会发现写操作之后读回来结果错误，发现是默认线性映射的问题，全都加载到内核态，问题解决。

![image-20250508203722758](C:\Users\24463\AppData\Roaming\Typora\typora-user-images\image-20250508203722758.png)

```rust
//线性映射地址转换
pub const fn virt_to_phys(vaddr: VirtAddr) -> PhysAddr {
    pa!(vaddr.as_usize() - PHYS_VIRT_OFFSET)
}

pub const fn phys_to_virt(paddr: PhysAddr) -> VirtAddr {
    va!(paddr.as_usize() + PHYS_VIRT_OFFSET)
}

pub fn write(&mut self, buf: &[u8]) -> AxResult<usize> {
    let offset = if self.is_append {
        self.get_attr()?.size()
    } else {
        self.offset
    };
    //拷贝我们的OS所在的内核态，默认线性映射
    let buf = buf.to_vec();
    let node = self.access_node(Cap::WRITE)?;
    let write_len = node.write_at(offset, &&*buf)?;
    self.offset = offset + write_len as u64;
    Ok(write_len)
}
```

![image-20250508204203466](C:\Users\24463\AppData\Roaming\Typora\typora-user-images\image-20250508204203466.png)

## 编写shm系列syscall框架

通过iozone测试的核心系统调用，要支持多个进程共享内存进行读写。

参考了byteos的相关实现思路，但底层接口不同，需要进行改写。

- 给ProcessData添加成员变量注册shared_memory使用情况

- 添加全局SHARED_MEMORY_TABLE记录共享内存情况，为一个shmid和SharedMemoryBlock的BTreeMap

  ```rust
  pub struct SharedMemoryBlock {
      pub start_addr: PhysAddr,
      pub size: usize,
      pub deleted: Mutex<bool>,
  }
  
  pub static SHARED_MEMORY_TABLE: Mutex<BTreeMap<isize, Arc<SharedMemoryBlock>>> = Mutex::new(BTreeMap::new());
  
  pub struct MappedSharedMemoryData {
      pub key: isize,
      pub mem: Arc<SharedMemoryBlock>,
      pub start: usize,
      pub size: usize,
  }
  
  pub struct ProcessData {
      /// The command line arguments
      pub command_line: Mutex<Vec<String>>,
  
      // address space related are shared with all threads
      /// The virtual memory address space.
      pub addr_space: Arc<Mutex<AddrSpace>>,
      pub shared_memories: Mutex<Vec<MappedSharedMemoryData>>,
      /// The user heap bottom
      heap_bottom: AtomicUsize,
      /// The user heap top
      heap_top: AtomicUsize,
      // TODO: resource limits
      // TODO: signals
      // TODO: futex?
  }
  ```

  - sys_shmget和sys_shmat现在的功能实现类似于将mmap的分配和映射拆开。

## **熟悉底层接口**

通读了axmm和axalloc库的代码，对现在框架的内存分配和映射逻辑有了更深的认识。

```rust
pub struct ProcessData {
    /// The command line arguments
    pub command_line: Mutex<Vec<String>>,

    // address space related are shared with all threads
    /// The virtual memory address space.
    pub addr_space: Arc<Mutex<AddrSpace>>,
    pub shared_memories: Mutex<Vec<MappedSharedMemoryData>>,
    /// The user heap bottom
    heap_bottom: AtomicUsize,
    /// The user heap top
    heap_top: AtomicUsize,
    // TODO: resource limits
    // TODO: signals
    // TODO: futex?
}
//...   
let current = current_process_data();
let mut aspace = current.addr_space.lock();
aspace.map_alloc(
    start_addr,
    aligned_length,
    permission_flags.into(),
    populate,
)?;
//...

pub fn map_alloc(
    &mut self,
    start: VirtAddr,
    size: usize,
    flags: MappingFlags,
    populate: bool,
) -> AxResult {
    self.validate_region(start, size)?;

    let area = MemoryArea::new(start, size, flags, Backend::new_alloc(populate));
    self.areas
        .map(area, &mut self.pt, false)
        .map_err(mapping_err_to_ax_err)?;
    Ok(())
}
//...
    pub fn map(
        &mut self,
        area: MemoryArea<B>,
        page_table: &mut B::PageTable,
        unmap_overlap: bool,
    ) -> MappingResult {
		//实际分配与映射
        area.map_area(page_table)?;
        //添加注册
        assert!(self.areas.insert(area.start(), area).is_none());
        Ok(())
    }
//...
    pub(crate) fn map_area(&self, page_table: &mut B::PageTable) -> MappingResult {
        self.backend
            .map(self.start(), self.size(), self.flags, page_table)
            .then_some(())
            .ok_or(MappingError::BadState)
}
//...
    fn map(&self, start: VirtAddr, size: usize, flags: MappingFlags, pt: &mut PageTable) -> bool {
        match *self {
            Self::Linear { pa_va_offset } => Self::map_linear(start, size, flags, pt, pa_va_offset),
            Self::Alloc { populate } => Self::map_alloc(start, size, flags, pt, populate),
        }
    }
//...
    pub(crate) fn map_alloc(
        start: VirtAddr,
        size: usize,
        flags: MappingFlags,
        pt: &mut PageTable,
        populate: bool,
    ) -> bool {
        debug!(
            "map_alloc: [{:#x}, {:#x}) {:?} (populate={})",
            start,
            start + size,
            flags,
            populate
        );
        if populate {
            // allocate all possible physical frames for populated mapping.
            for addr in PageIter4K::new(start, start + size).unwrap() {
                if let Some(frame) = alloc_frame(true) {
                    if let Ok(tlb) = pt.map(addr, frame, PageSize::Size4K, flags) {
                        tlb.ignore(); // TLB flush on map is unnecessary, as there are no outdated mappings.
                    } else {
                        return false;
                    }
                }
            }
        } else {
            // create mapping entries on demand later in `handle_page_fault_alloc`.
        }
        true
    }

fn alloc_frame(zeroed: bool) -> Option<PhysAddr> {
    let vaddr = VirtAddr::from(global_allocator().alloc_pages(1, PAGE_SIZE_4K).ok()?);
    if zeroed {
        unsafe { core::ptr::write_bytes(vaddr.as_mut_ptr(), 0, PAGE_SIZE_4K) };
    }
    let paddr = virt_to_phys(vaddr);
    Some(paddr)
}

```



## 




## For next week

- `shm`系列`syscall`实现
- `sys_shmget`
- `sys_shmat`
- `sys_shmctl`
