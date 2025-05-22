# OS LAB Report - week14

## `shm`支持

不参考或移植他人仓库的情况下独立完成了`shared_memory`的真实现。

```rust
Sysno::shmget => sys_shmget(tf.arg0() as _, tf.arg1() as _, tf.arg2() as _),
Sysno::shmat => sys_shmat(tf.arg0() as _, tf.arg1() as _, tf.arg2() as _),
Sysno::shmctl => sys_shmctl(tf.arg0() as _, tf.arg1() as _, tf.arg2() as _),
Sysno::shmdt => sys_shmdt(tf.arg0() as _),
```

并且相对`ByteOS`实现体现出以下优势：

- 实现更加正确，`ByteOS`实现存在`Bug`（发现后已经联系作者取得认可），无法正确释放共享内存段，我们的实现可以正确自动释放。

- 功能更为齐全，实现了`ByteOS`未实现的`syscall`，支持`ByteOS`未支持的参数，进行了`ByteOS`没有进行的检查和校验。

- 设计更加优雅，取消了不必要的成员变量，`Drop`的逻辑更加简洁自然。

  ```rust
  //ByteOS实现
  pub struct SharedMemory {
      pub trackers: Vec<Arc<FrameTracker>>,
      pub deleted: Mutex<bool>,
  }
  
  pub static SHARED_MEMORY: Mutex<BTreeMap<usize, Arc<SharedMemory>>> = Mutex::new(BTreeMap::new());
  
  pub struct MapedSharedMemory {
      pub key: usize,
      pub mem: Arc<SharedMemory>,
      pub start: usize,
      pub size: usize,
  }
  
  impl Drop for MapedSharedMemory {
      fn drop(&mut self) {
          // 这里的引用计数应该是2，设置为1永远不会触发物理内存释放
          // sys_shmctl在IPC_RMID参数下的行为就是将deleted成员变量设置为true
          if Arc::strong_count(&self.mem) == 1 && *self.mem.deleted.lock() == true {
              SHARED_MEMORY.lock().remove(&self.key);
          }
      }
  }
  
  pub struct ProcessControlBlock {
  	//...
      pub shms: Vec<MapedSharedMemory>,
  	//...
  }
  
  
  ```

  `Rust`对于结构体的`Drop`机制中，先`Drop`结构体本身才会`Drop`成员变量(参考[Drop 释放资源 - Rust语言圣经(Rust Course)](https://course.rs/advance/smart-pointer/drop.html))，所以针对某个具体的共享内存段（对应一个`SharedMemory`结构体），在`Drop`最后一个映射到它的`MapedSharedMemory`时，强引用会有两个（一个是自己`MapedSharedMemory`结构体中随后要释放的成员变量，另一个是 `SHARED_MEMORY`全局表单中的引用），应该设置为`2`。

  ```rust
  pub struct SharedMemory {
      /// The key of the shared memory segment
      pub key: u32,
      /// Virtual kernel address of the shared memory segment
      pub addr: usize,
      /// Page count of the shared memory segment
      pub page_count: usize,
  }
  
  impl Drop for SharedMemory {
      fn drop(&mut self) {
          let allocator = global_allocator();
          allocator.dealloc_pages(self.addr, self.page_count);
      }
  }
  
  pub struct SharedMemoryManager {
      mem_map: Mutex<BTreeMap<u32, Arc<SharedMemory>>>,
      next_key: AtomicU32,
  }
  
  impl SharedMemoryManager {
  	//...
      pub fn delete(&self, key: u32) -> bool {
           // sys_shmctl在IPC_RMID参数下的行为就是将对应的SharedMemory从全局的SHARED_MEMORY_MANAGER中remove
          self.mem_map.lock().remove(&key).is_some()
      }
  }
  
  pub static SHARED_MEMORY_MANAGER: SharedMemoryManager = SharedMemoryManager::new();
  
  pub struct ProcessData {
  	//方便sys_shmdt找到对应的SharedMemory，其参数是一个虚拟地址
      /// Shared memory
      pub shared_memory: Mutex<BTreeMap<VirtAddr, Arc<SharedMemory>>>,
  }
  ```

之所以`ByteOS`需要手动判定强引用次数就是因为全局表单中有一个一直没有释放的强引用，如果在`sys_shmctl`将一个共享内存块设置为待删除时直接将其从全局表单中移除就可以优雅地实现自动内存释放，也不再需要维护`SharedMemory`的`deleted`成员变量。

同时还有一个好处是：被标记为待删除的共享内存段不应还能够被`sys_shmat`映射，我们直接删除的做法可以让全局表单中查不到这个共享内存段，从而杜绝了这种行为，但`ByteOS`中仍然保留，所以需要在`shmget`中进行手动判断`deleted`来决定查到的共享内存段是否可以映射，更为麻烦。事实上`ByteOS`中也没有进行这个校验，这意味着`sys_shmctl`中标记为待删除的共享内存段依然可以在`sys_shmat`中映射。

```rust
//ByteOS
pub async fn sys_shmat(&self, shmid: usize, shmaddr: usize, shmflg: usize) -> SysResult {
       //...
    	//只检查了是否能查到这一块共享内存，而没有检查是否deleted待删除
        let trackers = SHARED_MEMORY.lock().get(&shmid).cloned();
        if trackers.is_none() {
            return Err(Errno::ENOENT);
        }
		//...
        Ok(vaddr.raw())
    }

```

同时我们相对`ByteOS`还多实现了`sys_shmdt`，支持进程中途解绑一段共享内存，而`ByteOS`只支持在进程结束后释放`ProcessControlBlock`时才能解绑，我们的实现可以更及时地释放内存资源。

特别地，我们发现由于Loongarch的内存布局和其他架构不同，需要给其分配更大的内存才足够通过测试。
## For next week

- 重构和精简现在的代码结构—减少反复嵌套调用，位置混乱等现象

- 尽可能多地把之前为了通过测例用的伪实现变为真实现—减少`syscall`桩函数的使用

- 简单了解和测试`LTP`测例

- 编写期末文档

  
