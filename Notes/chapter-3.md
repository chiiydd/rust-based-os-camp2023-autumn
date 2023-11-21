---
typora-copy-images-to: ./images
---

![锯齿螈多道程序操作系统 -- Multiprog OS总体结构](https://rcore-os.cn/rCore-Tutorial-Book-v3/_images/jcy-multiprog-os-detail.png)

> 通过上图，大致可以看出Qemu把包含多个app的列表和MultiprogOS的image镜像加载到内存中，RustSBI（bootloader）完成基本的硬件初始化后，跳转到MultiprogOS起始位置，MultiprogOS首先进行正常运行前的初始化工作，即建立栈空间和清零bss段，然后通过改进的 AppManager 内核模块从app列表中把所有app都加载到内存中，并按指定顺序让app在用户态一个接一个地执行。app在执行过程中，会通过系统调用的方式得到MultiprogOS提供的OS服务，如输出字符串等。

这是第二章实现的批处理系统，只实现了一些简单的系统调用，APPManager只能一个个按照顺序执行APP,前一个APP完成后才能调度后一个app，CPU的利用率并不高。然后为了提高效率，第三章的腔骨龙分时多任务操作系统改进了 Trap_handler和AppManager模块，提供任务调度功能，当前的任务时间片用完后会切换其他任务执行。

![始初龙协作式多道程序操作系统 -- CoopOS总体结构](https://rcore-os.cn/rCore-Tutorial-Book-v3/_images/more-task-multiprog-os-detail.png)

第三章需要实现的一个获取当前执行任务的任务信息的系统调用。

- 我是首先在任务控制块中加入了两个字段，一个是任务的创建事件，一个是任务信息。

```rust
pub struct TaskControlBlock {
	——————————————————————
    /// task create time
    pub create_time:usize,

    /// taskinfo 
    pub task_info: TaskInfo,
    
}
```



- 然后为TaskInfo实现了更改状态、更改时间、更改系统调用的方法

  - pub fn change_status(&mut self,status:TaskStatus)
  - pub fn count_syscall_times(&mut self,syscall_number:usize,num:u32)
  - pub fn update_time(&mut self,ctime:usize )

- 然后把其封装在TaskManager中，并且再封装一个同名函数来使得TASKMANAGER来获取当前任务调用相应的函数。

- 然后在syscall中，调用update_taskinfo_time和update_taskinfo_syscalltime来更新时间和系统调用次数。

- 最后便是实现sys_taskinfo的系统调用

  - 调用get_current_taskinfo()获取当前任务信息
  - 利用传入的taskinfo指针进行赋值。

  
