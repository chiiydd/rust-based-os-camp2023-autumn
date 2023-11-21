---
typora-copy-images-to: ./images
---

# records

## 2023-11-8

今天学习了unikernel的概念，arceos的大致实现情况。

**Unikernel形态**

- 单应用
- 单地址空间:应用与OS共用一个地址空间，不分用户空间和内核空间。
- 单特权级:只有一个运行privilege level，不分用户态和内核态。

总结来说，Unikernel形态下的操作系统，应用和操作系统是一体的，操作系统退化成一个组件库，为上层的应用提供服务。由于不区分用户空间和内核空间，也不区分用户态和核心态，减少了内核切换的开销，提升应用的运行效率。

ArceOS的层次

<img src="C:\post-graduate\rcore\Notes\images\image-20231111145324389.png" alt="image-20231111145324389" style="zoom:50%;" />



下面是ArceOS的一个启动过程，运行一个hello_world程序。



<img src="C:\post-graduate\rcore\Notes\images\image-20231111145501003.png" alt="image-20231111145501003" style="zoom:50%;" />



## 2023-11-09

今天主要是熟悉arceos的代码，尝试做练习1和练习2

### 练习1

首先练习要求打印彩色输出，我想到本来的日志输出就是彩色的，因此我可以先参考axlog模块进行修改。

起初我以为是修改putchar函数

```rust
// modules/axhal/src/platform/riscv64_qemu_virt/console.rs


/// Writes a byte to the console.
pub fn putchar(c: u8) {
    #[allow(deprecated)]
    sbi_rt::legacy::console_putchar(c as usize);
    
}

/// Reads a byte from the console, or returns [`None`] if no input is available.
pub fn getchar() -> Option<u8> {
    #[allow(deprecated)]
    match sbi_rt::legacy::console_getchar() as isize {
        -1 => None,
        c => Some(c as u8),
    }
}

```

但是，好像似乎没有什么可以修改的地方，查阅了一下sbi的文档好像也没有相关的东西。

而且axlog的输出似乎是跟系统输出是分离的

```rust
/// Prints the formatted string to the console.
pub fn print_fmt(args: fmt::Arguments) -> fmt::Result {
    use spinlock::SpinNoIrq; // TODO: more efficient
    static LOCK: SpinNoIrq<()> = SpinNoIrq::new(());

    let _guard = LOCK.lock();
    Logger.write_fmt(args)
}

#[doc(hidden)]
pub fn __print_impl(args: fmt::Arguments) {
    print_fmt(args).unwrap();
}
```



然后重新回到hellowold中，跳转回去查看println!的实现

```rust
//  arceos/ulib/axstd/src/macros.rs
/// Prints to the standard output, with a newline.
#[macro_export]
macro_rules! println {
    () => { $crate::print!("\n") };
    ($($arg:tt)*) => {
        $crate::io::__print_impl(format_args!("\u{1B}[33m{}\u{1B}[m",format_args!("{}\n", format_args!($($arg)*))));
    }
}

// ulib/axstd/src/io/stdio.rs
#[doc(hidden)]
pub fn __print_impl(args: core::fmt::Arguments) {
    if cfg!(feature = "smp") {
        // synchronize using the lock in axlog, to avoid interleaving
        // with kernel logs
        arceos_api::stdio::ax_console_write_fmt(args).unwrap();
    } else {
        stdout().lock().write_fmt(args).unwrap();
    }
}


```

最后，在println的宏中添加字符得以实现

## 2023-11-10

### 练习2

一开始有点懵，看着rust官方的HashMap实现有三千行，在移植的过程中，有很多的相关的库。



有一个疑问：官方std库也是用的hashbrown库中实现的HashMap，进行重新封装

因此有一个偷懒的做法

```rust
//ulib/axstd/src/collections/mod.rs

pub use hashbrown::HashMap
```



但是不能这样做，在一个个移植的时候发现有很多包内其他的mod，可能有点复杂。

然后在群聊中看到老师的提示:只需要移植 用到的函数就可以了

也就是

- 初始化 new

- 嵌入 insert

- 迭代器 Iterator

  因此涉及这三块的内容并不多

  ```rust
  use hashbrown::hash_map as base;
  use spinlock::SpinNoIrq;
  #[allow(deprecated)]
  use core::hash::{BuildHasher, Hash, Hasher, SipHasher13};
  
  pub struct HashMap<K, V, S = RandomState> {
      base: base::HashMap<K, V, S>,
  
  }
  
  impl<K, V> HashMap<K, V, RandomState> {
  
      #[inline]
      #[must_use]
      pub fn new() -> HashMap<K, V, RandomState> {
          Default::default()
      }
  
  }
  impl <K,V,S> HashMap<K,V,S>{
      #[inline]
      pub const fn with_hasher(hash_builder: S) -> HashMap<K, V, S> {
          HashMap { base: base::HashMap::with_hasher(hash_builder) }
      }
  }
  
  impl<K, V, S> Default for HashMap<K, V, S>
  where
      S: Default,
  {
      /// Creates an empty `HashMap<K, V, S>`, with the `Default` value for the hasher.
      #[inline]
      fn default() -> HashMap<K, V, S> {
          HashMap::with_hasher(Default::default())
      }
  
      
  }
  
  
  pub struct RandomState {
      k0: u64,
      k1: u64,
  }
  
  impl RandomState {
      #[inline]
      #[allow(deprecated)]
      // rand
      #[must_use]
      pub fn new() -> RandomState {
  
  
  
          //TODO: get random numbers: k0 and k1.
          let ret =random();
          // test k0&k1
          // let k0=(ret>>64) as u64;
          // let k1=(ret&0x0000_0000_ffff_ffff_ffff_ffff )as u64;
          // println!("{}:{}",k0,k1);
          RandomState {
              k0:(ret>>64) as u64,
              k1:(ret&0x0000_0000_ffff_ffff_ffff_ffff )as u64,
          }
      }
  }
  
  impl BuildHasher for RandomState {
      type Hasher = DefaultHasher;
      #[inline]
      #[allow(deprecated)]
      fn build_hasher(&self) -> DefaultHasher {
          DefaultHasher(SipHasher13::new_with_keys(self.k0, self.k1))
      }
  }
  
  #[allow(deprecated)]
  #[derive(Clone, Debug)]
  pub struct DefaultHasher(SipHasher13);
  
  impl DefaultHasher {
      /// Creates a new `DefaultHasher`.
      #[inline]
      #[allow(deprecated)]
      #[must_use]
      pub const fn new() -> DefaultHasher {
          DefaultHasher(SipHasher13::new_with_keys(0, 0))
      }
  }
  
  impl Default for DefaultHasher {
      /// Creates a new `DefaultHasher` using [`new`].
      /// See its documentation for more.
      #[inline]
      fn default() -> DefaultHasher {
          DefaultHasher::new()
      }
  }
  
  impl Hasher for DefaultHasher {
      // The underlying `SipHasher13` doesn't override the other
      // `write_*` methods, so it's ok not to forward them here.
  
      #[inline]
      fn write(&mut self, msg: &[u8]) {
          self.0.write(msg)
      }
  
      #[inline]
      fn write_str(&mut self, s: &str) {
          self.0.write_str(s);
      }
  
      #[inline]
      fn finish(&self) -> u64 {
          self.0.finish()
      }
  }
  
  impl Default for RandomState {
      /// Constructs a new `RandomState`.
      #[inline]
      fn default() -> RandomState {
          RandomState::new()
      }
  }
  
  
  impl <K,V,S> HashMap<K,V,S> {
      #[rustc_lint_query_instability]
      pub fn iter(&self) -> Iter<'_, K, V> {
          Iter { base: self.base.iter() }
      }
  }
  
  pub struct Iter<'a, K: 'a, V: 'a> {
      base: base::Iter<'a, K, V>,
  }
  
  
  impl<'a, K, V> Iterator for Iter<'a, K, V> {
      type Item = (&'a K, &'a V);
  
      #[inline]
      fn next(&mut self) -> Option<(&'a K, &'a V)> {
          self.base.next()
      }
      #[inline]
      fn size_hint(&self) -> (usize, Option<usize>) {
          self.base.size_hint()
      }
  }
  
  
  impl<K, V, S> HashMap<K, V, S>
  where
      K: Eq + Hash,
      S: BuildHasher,
  {
      #[inline]
      pub fn insert(&mut self, k: K, v: V) -> Option<V> {
          self.base.insert(k, v)
      }
  }
  
  
  static PARK_MILLER_LEHMER_SEED:SpinNoIrq<u32>=SpinNoIrq::new(0);
  // 2^32 -1=4294967295
  const RAND_MAX: u64 = 4_294_967_295;
  // get a radom u128 number.
  pub fn random()->u128{
      let mut seed =PARK_MILLER_LEHMER_SEED.lock();
      if *seed==0{
          *seed=arceos_api::time::ax_current_time().as_micros() as u32 ;
      }
      let mut ret:u128=0;
  
      for _ in 0..4{
          *seed = ((u64::from(*seed) * 48271) % RAND_MAX) as u32;
          ret= (ret << 32) | (*seed as u128);
      }
      ret 
  
  }
  
  ```

  

## 2023-11-11

阅读原先的modules/axalloc/src/lib.rs代码，还有BitmapAllocator、buddyAllocator、slabAllocator代码，理解其中的含义。



## 2023-11-12

初步尝试实现EarlyAllocator，初步的想法是通过复用 BitmapAllocator和上面的其中一个byteAllocator。在看以下源码的时候，我以为是byteAllocator申请的内存需要先在pageAllocator中申请之后才能申请。

```rust
    pub fn alloc(&self, layout: Layout) -> AllocResult<NonNull<u8>> {
        // simple two-level allocator: if no heap memory, allocate from the page allocator.
        let mut balloc = self.balloc.lock();
        loop {
            if let Ok(ptr) = balloc.alloc(layout) {
                return Ok(ptr);
            } else {
                let old_size = balloc.total_bytes();
                let expand_size = old_size
                    .max(layout.size())
                    .next_power_of_two()
                    .max(PAGE_SIZE);
                let heap_ptr = self.alloc_pages(expand_size / PAGE_SIZE, PAGE_SIZE)?;
                debug!(
                    "expand heap memory: [{:#x}, {:#x})",
                    heap_ptr,
                    heap_ptr + expand_size
                );
                balloc.add_memory(heap_ptr, expand_size)?;
            }
        }
    }
```

因此进度迟迟卡住了。

## 2023-11-13

今天也是继续错误的思路的一天，因为我以为  byteAllocator中的heap和pageAllocator中的BitAllocUsed是“底层”的内存分配机制，我的EarlyAllocator中需要有该字段。

然后就在思考如何利用BitAllocUsed实现从高地址到低地址进行分配页。。。。



## 2023-11-14

终于理清思路了，实际上底层的内存是一次性分配好了的(老师简化过程了)，我想的复杂了，Allocator的作用是在用户程序申请内存时返回正确的地址空间。





## 2023-11-15



练习4，主要是理解DTB的数据结构类型



练习3的一个问题，因为地址空间未对齐，运行练习5的时候报错了。

![image-20231115160810402](C:\post-graduate\rcore\Notes\images\image-20231115160810402.png)



## 2023-11-16

练习5  详细查看 Scheduler的几个函数的作用

```rust
//! Various scheduler algorithms in a unified interface.
//!
//! Currently supported algorithms:
//!
//! - [`FifoScheduler`]: FIFO (First-In-First-Out) scheduler (cooperative).
//! - [`RRScheduler`]: Round-robin scheduler (preemptive).
//! - [`CFScheduler`]: Completely Fair Scheduler (preemptive).

#![cfg_attr(not(test), no_std)]
#![feature(const_mut_refs)]

mod cfs;
mod fifo;
mod round_robin;

#[cfg(test)]
mod tests;

extern crate alloc;

pub use cfs::{CFSTask, CFScheduler};
pub use fifo::{FifoScheduler, FifoTask};
pub use round_robin::{RRScheduler, RRTask};

/// The base scheduler trait that all schedulers should implement.
///
/// All tasks in the scheduler are considered runnable. If a task is go to
/// sleep, it should be removed from the scheduler.
pub trait BaseScheduler {
    /// Type of scheduled entities. Often a task struct.
    type SchedItem;

    /// Initializes the scheduler.
    fn init(&mut self);

    /// Adds a task to the scheduler.
    fn add_task(&mut self, task: Self::SchedItem);

    /// Removes a task by its reference from the scheduler. Returns the owned
    /// removed task with ownership if it exists.
    ///
    /// # Safety
    ///
    /// The caller should ensure that the task is in the scheduler, otherwise
    /// the behavior is undefined.
    fn remove_task(&mut self, task: &Self::SchedItem) -> Option<Self::SchedItem>;

    /// Picks the next task to run, it will be removed from the scheduler.
    /// Returns [`None`] if there is not runnable task.
    fn pick_next_task(&mut self) -> Option<Self::SchedItem>;

    /// Puts the previous task back to the scheduler. The previous task is
    /// usually placed at the end of the ready queue, making it less likely
    /// to be re-scheduled.
    ///
    /// `preempt` indicates whether the previous task is preempted by the next
    /// task. In this case, the previous task may be placed at the front of the
    /// ready queue.
    fn put_prev_task(&mut self, prev: Self::SchedItem, preempt: bool);

    /// Advances the scheduler state at each timer tick. Returns `true` if
    /// re-scheduling is required.
    ///
    /// `current` is the current running task.
    fn task_tick(&mut self, current: &Self::SchedItem) -> bool;

    /// set priority for a task
    fn set_priority(&mut self, task: &Self::SchedItem, prio: isize) -> bool;
}

```



主要是 task_tick是返回是可以重新调度，主要是修改这个逻辑。

FifoScheduler重的这个函数是直接返回false的，因为原先是不可剥夺的。因此我们可以仿照RRScheduler中增加一个时间片的计数器，如果时间片用完了返回true，时间片没用完返回false。



在modules/axtask/Cargo.toml中启用 preemt特征

```
[features]
default = ["preempt"]
```









`dd`命令用于读取、转换并输出数据。它可以从标准输入或文件中读取数据，根据指定的格式来转换数据，再输出到文件、设备或标准输出。`dd`命令在Linux系统中非常有用，可以用于备份磁盘、拷贝文件、转换数据等。`dd`命令的语法与大多数其他Linux命令不同，但是它的基本用法非常简单。默认情况下，`dd`命令从标准输入读取数据，并将数据写入标准输出。但是，我们可以使用`if`和`of`命令行选项来指定输入和输出。以下是一些常用的`dd`命令选项：

- `if=文件名`：输入文件名，默认为标准输入。

- `of=文件名`：输出文件名，默认为标准输出。

- `ibs=bytes`：一次读入`bytes`个字节，即指定一个块大小为`bytes`个字节。

- `obs=bytes`：一次输出`bytes`个字节，即指定一个块大小为`bytes`个字节。

- `bs=bytes`：同时设置读入/输出的块大小为`bytes`个字节。

- `skip=blocks`：从输入文件开头跳过`blocks`个块后再开始复制。

- `seek=blocks`：从输出文件开头跳过`blocks`个块后再开始复制。

- `count=blocks`：仅拷贝`blocks`个块，块大小等于`ibs`指定的字节数。

- ```
  conv=<关键字>
  ```

  ：关键字可以有以下11种：

  - `conversion`：用指定的参数转换文件。
  - `ascii`：转换ebcdic为ascii。
  - `ebcdic`：转换ascii为ebcdic。
  - `ibm`：转换ascii为alternate ebcdic。
  - `block`：把每一行转换为长度为`cbs`，不足部分用空格填充。
  - `unblock`：使每一行的长度都为`cbs`，不足部分用空格填充。
  - `lcase`：把大写字符转换为小写字符。
  - `ucase`：把小写字符转换为大写字符。
  - `swap`：交换输入的每对字节。
  - `noerror`：出错时不停止。
  - `notrunc`：不截短输出文件。
  - `sync`：将每个输入块填充到`ibs`个字节，不足部分用空（NUL）字符补齐。



## 2023-11-17

第二周练习

发现镜像文件得对齐32M，否则qemu无法加载

![image-20231117232445605](C:\post-graduate\rcore\Notes\images\image-20231117232445605.png)
