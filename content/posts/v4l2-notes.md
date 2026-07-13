---
title: "V4L2 驱动框架深度分析"
date: 2026-07-13
draft: false
tags: ["Linux内核", "V4L2", "驱动开发", "视频设备"]
summary: "深入分析 Linux V4L2 驱动框架，涵盖数据结构、注册流程、ioctl 调用链、videobuf2 缓冲区管理。"
---
# v4l2

<div>
https://blog.csdn.net/u013904227/article/details/80718831
</div>
https://blog.csdn.net/qq_42732669/article/details/139580399
</div>

# 概述

由于硬件的复杂性，同时 v4l2 驱动要支持 IC 模块进行音视频的混合编解码操作，v4l2 的驱动变得异常复杂。

老旧的 v4l2 框架仅限于通过 video_device 结构体创建 v4l 设备节点和 video_buf 来处理视频数据。

这意味着所有的驱动都必须对设备实例进行设置并将其映射到子设备上。有些时候这些操作步骤十分复杂，很难正确完成，并且有些驱动程序从来没有正确的按照这些操作步骤编写。由于缺少一个框架，有很多通用代码就没有办法被重构，从而导致这部分代码被重复编写，效率比较低下。

所以该框架抽象构建了所以驱动都需要的代码并封装为一个个模块，简化了设备驱动通用代码的重构。

V4L2 只做三件事：

**1. 管理设备节点**
提供一个 `/dev/videoX`，让应用能 open/close/ioctl。这背后的工作是次设备号分配、cdev 注册、sysfs 属性。

**2. 标准化控制接口**
定义了一百多个 VIDIOC\_\* 命令，让应用用同一套 API 控制不同硬件。不管摄像头是 USB 的、MIPI 的、还是并口的，调亮度都用同一个 `VIDIOC_S_CTRL`​。差异被隔离在驱动层的 `vidioc_*` 回调里。

**3. 搬运数据**  
提供 videobuf2 框架管理 DMA 缓冲区的生命周期：分配、排队、完成通知、内存映射。它只管把数据从硬件搬到内存、再从内存交给应用。数据到了应用手里之后怎么处理，V4L2 不管。

# 框架

从应用的角度来看如何使用v4l2

![v2-7b72825dbb76f120f821a9ab0c7905da_r](/images/v4l2/v2-7b72825dbb76f120f821a9ab0c7905da_r-20260710185034-lap20xj.jpg)

先看正常的硬件拓扑结构，才能深入理解

![image](/images/v4l2/image-20260710184609-4pjbkn8.png)

通常一个camera的模组如图所示，通常包括Lens、Sensor、CSI接口等，其中CSI接口用于视频数据的传输；

SoC的Mipi接口对接Camera，并通过I2C/SPI控制camera模组；

Camera模组中也可以包含ISP模块，用于对图像进行处理，有的SoC中也集成了ISP的IP，接收camera的raw数据后，进行图像处理；

对硬件设备进行抽象

以v4l2_device代表整个输入设备，以v4l2_subdev代表子模块，比如CSI，sensor。

![v2-69ea1f9d8bde110a0d34dcf13d5c68d3_r](/images/v4l2/v2-69ea1f9d8bde110a0d34dcf13d5c68d3_r-20260710185916-2p3who7.jpg)

可以很容易的理解v4l2子系统的数据结构了

# 数据结构

## struct v4l2_device

该结构体是对整个输入设备的抽象，将各个子设备联系在一起，起到纽带的作用。通常会嵌入在其他结构体中提供v4l2框架功能，在实现子设备驱动中通常会嵌入驱动自己定义的结构体中。

## struct v4l2_subdev

v4l2\_subdev是对子设备的抽象，驱动自己定义的结构体放私有数据，该结构体中包含的struct v4l2\_subdev\_ops是一个完备的操作函数集，用于对接各种不同的子设备，比如video、audio、sensor等，同时还有一个核心的函数集struct v4l2\_subdev\_core\_ops，提供更通用的功能。子设备驱动根据设备特点实现该函数集中的某些函数即可；

## struct media_device

一个 V4L2 device 下属可能有非常多同类型的子设备（两个或者多个 sensor、ISP 等），那么在设备运行的时候我怎么知道我的数据流需要用到哪一个类型的哪一个子设备呢。这个时候就轮到 media_device 出手了，它为这一坨的子设备建立一条虚拟的连线，建立起来一个运行时的 pipeline（管道），并且可以在运行时动态改变、管理接入的设备。

## struct video_device

video\_device的作用是向vfs注册节点供用户层使用

```填写video_device
video_device 注册后
  → cdev_add 把 video_device->cdev 注册到 VFS
    → 用户就能 open("/dev/video0")
      → VFS 通过主设备号 81 找到 cdev
        → cdev 的 ops 是 v4l2_fops
          → v4l2_fops.open = v4l2_open
            → 通过 video_devdata(file) 找到 video_device
              → 再找到驱动的 ioctl_ops 和 vb2_queue
```

用于向系统注册字符设备节点，以便用户空间可以进行交互，包括各类设置以及数据buffer的获取等，在该结构体中也能看到struct v4l2\_ioctl\_ops和struct vb2\_queue结构体字段，这些与上文中的应用层代码编写息息相关；可以内嵌在其他结构体中，以便提供用户层交互的功能

# 注册流程分析

以airspy.c为例

在probe函数中调用video_register_device进行注册

![image](/images/v4l2/image-20260711151919-b7lggbh.png)

跳转到__video_register_device

![image](/images/v4l2/image-20260711152017-4bkiccr.png)

在__video_register_device进行注册

注册字符驱动设备

‍

![image](/images/v4l2/image-20260711153057-ngd0sen.png)

注册/sys下的class

![image](/images/v4l2/image-20260711153200-gyqhsob.png)

‍

|部分|给谁看|函数|效果|
| ----------| --------| ------| ----------------------------------------------------------|
|字符设备|**用户层**|`video_register_device`|`/dev/video0`出现，应用可以 open/ioctl|
|设备注册|**内核**|`v4l2_device_register`|关联到父设备，挂 subdev 链表，提供 ctrl\_handler 继承|

![image](/images/v4l2/image-20260711155626-5e7ocvi.png)

## 函数表分析

查看airspy结构体

```
struct airspy {
    struct video_device vdev;         // ← 里面藏了 fops + ioctl_ops
    struct v4l2_device v4l2_dev;      // ← 里面藏了 ctrl_handler 指针
    struct vb2_queue vb_queue;        // ← 里面藏了 ops + mem_ops
    struct v4l2_ctrl_handler hdl;     // ← 里面藏了 ctrl_ops
    ...
};
```

在probe函数中

![image](/images/v4l2/image-20260711162706-gjexrj3.png)

![image](/images/v4l2/image-20260711162748-ftrq6zv.png)

填充了airspy自定义的结构体的video_device，包括fops，ioctl_ops

```ops填充
struct airspy {
    vdev ───── video_device
                ├── .fops       → &airspy_fops (来自模板拷贝)
                │     ├── .open    = v4l2_fh_open      ← 核心层
                │     ├── .read    = vb2_fop_read       ← vb2 框架
                │     ├── .mmap    = vb2_fop_mmap       ← vb2 框架
                │     └── .unlocked_ioctl = video_ioctl2 ← 核心层
                │
                └── .ioctl_ops  → &airspy_ioctl_ops (来自模板拷贝)
                      ├── .vidioc_querycap      = airspy_querycap         ← 驱动
                      ├── .vidioc_s_fmt_sdr_cap = airspy_s_fmt_sdr_cap    ← 驱动
                      ├── .vidioc_qbuf          = vb2_ioctl_qbuf          ← vb2 框架
                      └── .vidioc_streamon      = vb2_ioctl_streamon      ← vb2 框架

    vb_queue ── vb2_queue
                ├── .ops       → &airspy_vb2_ops
                │     ├── .queue_setup     = airspy_queue_setup     ← 驱动
                │     ├── .buf_queue       = airspy_buf_queue       ← 驱动
                │     ├── .start_streaming = airspy_start_streaming ← 驱动
                │     └── .stop_streaming  = airspy_stop_streaming  ← 驱动
                │
                └── .mem_ops   → &vb2_vmalloc_memops (框架提供，驱动选)

    hdl ─────── v4l2_ctrl_handler
                └── .ops      → &airspy_ctrl_ops
                      └── .s_ctrl = airspy_s_ctrl ← 驱动
}
```

注册的总体概览如下

![image](/images/v4l2/image-20260711164326-1zhhryg.png)

驱动结构体中内嵌 `struct video_device`​ 和 `struct vb2_queue`​，驱动实现 `ioctl_ops`​、`vb2_ops`​、`ctrl_ops`​ 这些硬件相关的回调，通过模板填充 `vdev.fops`​（指向框架提供的函数），最终通过 `video_register_device` 注册。

`__video_register_device`​ 内部通过 `cdev_add`​ 注册字符设备（固定使用核心层的 `v4l2_fops`​），通过 `device_register`​ 在 `/sys/class/video4linux/`​ 下创建设备节点。用户空间的 open/read/write/ioctl 先进入 `v4l2_fops`​，经过核心层处理后，最终路由到驱动注册的 `ioctl_ops`​ 和 `vb2_ops` 回调。

完成注册后，应用层可以通过文件描述符来访问设备节点，通过ioctl接口完成

## ioctl的调用流程分析

用户层使用ioctl(fd,VIDIOC\_xxx)命令，会调用v4l2_ioctl

![image](/images/v4l2/image-20260711194549-aqocpny.png)

在函数内部，先检测注册的fops是否提供了`unlocked_ioctl` 函数

![image](/images/v4l2/image-20260711194723-s8cp4vu.png)

早年 Linux 的 `file_operations`​ 里只有 `.ioctl`​，内核调用它之前会持有 ​**BKL（Big Kernel Lock）** ——一把全局大锁，性能极差。

后来内核引入了 `.unlocked_ioctl`​，意思是：​**VFS 层不再帮你持有任何锁，你自己管自己的并发**。VFS 侧的语义就是"没有锁"。

所以 `.unlocked_ioctl` 这个名字是 VFS 层的 contract，不是 V4L2 内部的状态。

如果有`unlocked_ioctl`，VFS层会调用这个

![image](/images/v4l2/image-20260711200718-d3jai8a.png)

调用这个函数

![image](/images/v4l2/image-20260711201031-i3cd7i0.png)

![image](/images/v4l2/image-20260711201158-ohx7i4t.png)

涉及到两个函数video_usercopy和__video_do_ioctl

### 先说video_usercopy

它干的活很纯粹，就是 ​ **"把数据从用户态搬到内核态，调完再搬回去"** ：

**进入时做的事：**

1.判断命令是否有参数。如果没有数据方向的（`_IOC_NONE`），跳过复制。

2.如果有参数：

	参数 ≤ 128 字节 → 用栈上的 `sbuf[128]`，避免分配

	参数 \> 128 字节 → `kmalloc`​ 动态分配 `mbuf`

3.如果是写方向（`_IOC_WRITE`​）：`copy_from_user(parg, arg, n)` — 从用户空间拷进来

	这里有个优化：如果 cmd 在已知表中且标记了 `INFO_FL_CLEAR_MASK`，只复制前几个字段，后面清零

4.如果是只读方向：直接 `memset` 清零内核缓冲区，留给驱动去填充

5.处理数组参数（如 `VIDIOC_ENUM_FMT`​ 里嵌套的数组指针）：`check_array_args` → 再拷一次数组

**中间做的事：**

```
err = func(file, cmd, parg);   // 调用 __video_do_ioctl
```

**返回时做的事：**

1.数组参数写回用户空间：`copy_to_user(user_ptr, mbuf, array_size)`

2.普通参数写回用户空间：

```
if (_IOC_DIR(cmd) & _IOC_READ)
    copy_to_user((void __user *)arg, parg, _IOC_SIZE(cmd));
```

3.`kfree(mbuf)` 收尾

**一句话：**​**​`video_usercopy`​**​ **不关心 ioctl 是什么含义，它只负责把参数安全地跨过内核/用户边界。**

### 再看__video_do_ioctl

![image](/images/v4l2/image-20260711202432-yd1f24k.png)

根据次设备号拿到注册的ioctl_ops

![image](/images/v4l2/image-20260711202738-j7zznm9.png)

查命令表

![image](/images/v4l2/image-20260711202901-6l6r8o6.png)

命令表记录了IOCTL命令和对应的跳转函数

最后就是校验 + 分发

根据 `info->flags`，命令分为两种

|标志|含义|分发方式|
| ------| ----------------| ----------------------|
|`INFO_FL_FUNC`|有专用处理函数|直接调`info-&gt;u.func(file, fh, parg)`|
|`INFO_FL_STD`|标准 ioctl|查驱动`ioctl_ops`里对应的回调|

比如 `VIDIOC_QBUF`​ 是 `INFO_FL_FUNC`​，直接调 `v4l2_qbuf`​（内核自己实现了标准流程）。而 `VIDIOC_S_FMT`​ 走 `INFO_FL_STD`​，最后调 `ops->vidioc_s_fmt`（驱动自己的实现）。

‍

![image](/images/v4l2/image-20260711205310-hrf77vc.png)

![image](/images/v4l2/image-20260711221332-4abpyjc.png)

# 数据流videobuf2

videobuf2用于连接V4L2驱动层与用户空间层，提供数据交流的通道，它可以分配并管理视频帧数据。videobuf层实现了很多的ioctl函数，包括buffer的分配、入队、出队和数据流控制。

## 数据类型

不是所有的视频设备都使用同一种类型的 buffer，事实上，buffer 类型可以概括为三种：

buffer 在虚拟地址以及物理地址上面都是分散的。几乎所有的用户空间 buffer 都是这种类型，在内核空间中，这种类型的 buffer 并不总是能够满足需要，因为这要求硬件可以进行分散的 DMA 操作。  
物理地址分散，虚拟地址连续。也就是通过 vmalloc 分配的 buffer，换句话说很难使用 DMA 来操作这些buffer。  
物理地址连续（虚拟地址不关心）。在分段式系统上面分配这种类型的 buffer 是不可靠的（分段式-也即页式系统上面有可能在长时间运行之后内存出现大量碎片，从而导致连续的内存空间很难获得，所以说不可靠），但是简单 DMA 控制器只能够适用于这种类型的 buffer。也有分段式的 DMA 操作，不过那种需要用到 IOMMU 来实现。  
这三种类型的 buffer 对应的相关操作函数分别对应于 videobuf2-dma-sg.c,videobuf2-vmalloc.c,videobuf2-dma-contig.c。

## buffer的内核实现

`APP操作buffer的示意图如下：`

![image](/images/v4l2/image-20260712171502-zir9hed.png)

### `videobuffer2缓冲区结构体`

驱动程序初始化时，就构造了vb2_queue，这是"buffer的队列"，一开始里面没有"buffer"

APP调用ioctl VIDIOC_REQBUFS向驱动申请N个buffer

驱动程序分配n(n<=N)个vb2_buffer结构体，然后

对于普通摄像头，还分配一个vb2_plane结构体、vb2_vmalloc_buf结构体，最后分配存数据的buffer

对于多平面摄像头，给每个vb2_buffer分配多个"vb2_plane结构体、vb2_vmalloc_buf结构体、存数据的buffer"

#### vb2_queue

用于管理整个buffer系统

```c
struct vb2_queue {
    /* ① 配置（驱动填充，只读） */
    unsigned int            type;           // V4L2_BUF_TYPE_VIDEO_CAPTURE 等
    unsigned int            io_modes;       // 支持的 I/O 方式：VB2_MMAP | VB2_USERPTR | VB2_READ
    struct device           *dev;           // DMA 设备

    const struct vb2_ops    *ops;           // → 驱动回调
    const struct vb2_mem_ops *mem_ops;      // → 内存后端操作
    void                    *drv_priv;      // → 驱动私有数据 (airspy 里设成 s)
    unsigned int            buf_struct_size; // 驱动自定义 buffer 结构体大小
    u32                     min_buffers_needed; // 至少需要多少 buffer 才启动

    /* ② 内部状态（应用不可见） */
    struct vb2_buffer       *bufs[VB2_MAX_FRAME]; // ★ buffer 指针数组，按 index 索引
    unsigned int            num_buffers;    // 实际分配了多少个 buffer

    struct list_head        queued_list;    // 已入队但还没交给驱动的 buffer
    unsigned int            queued_count;
    atomic_t                owned_by_drv_count; // 当前在驱动手里的 buffer 数

    struct list_head        done_list;      // ★ 硬件完成、等应用 DQBUF 的 buffer
    spinlock_t              done_lock;
    wait_queue_head_t       done_wq;        // ★ DQBUF 阻塞等待的等待队列

    unsigned int            streaming:1;           // 是否 STREAMON 状态
    unsigned int            start_streaming_called:1; // start_streaming 已被调用
    unsigned int            error:1;
    unsigned int            waiting_for_buffers:1;
    unsigned int            last_buffer_dequeued:1;
};
```

在airspy.c中可以看到驱动填充的过程

![image](/images/v4l2/image-20260712184608-8vjfr2m.png)

核心设计思想：**​`bufs[]`​** ​ **数组做存储，三个链表做状态管理**。

#### vb2_buffer

将vb2_buffer看做对buffer的描述，而不是buffer本身

```c
struct vb2_buffer {
    struct vb2_queue        *vb2_queue;     // 所属 queue
    unsigned int            index;          // 在 bufs[] 中的索引
    unsigned int            type;           // buffer 类型
    unsigned int            memory;         // VB2_MEMORY_MMAP/USERPTR/DMABUF
    unsigned int            num_planes;     // plane 数量（YUV 可能有 3 个 plane）
    struct vb2_plane        planes[VB2_MAX_PLANES]; // ★ 每个 plane 的内存信息

    /* 内部使用 */
    enum vb2_buffer_state   state;          // ★ 当前状态（状态机核心）
    struct list_head        queued_entry;   // 挂在 queued_list 上的链表节点
    struct list_head        done_entry;     // 挂在 done_list 上的链表节点

    u64                     timestamp;      // 时间戳
};
```

**每个 buffer 通过** **​`queued_entry`​**​ **和** **​`done_entry`​**​ **两个链表节点，可以在不同队列间移动**，不需要拷贝数据。

##### 补充：状态机

状态机就是把一件事分成几个过程，每个过程对应一个状态，状态与状态之间的转换通过事件来触发

一个状态机可以分为**状态，事件，动作，迁移条件**

**状态**：一个系统在某一时刻所存在的稳定的工作情况，系统在整个工作周期中可能有多个状态。例如一部电动机共有正转、反转、停转这 3 种状态。

**事件**：发生事件使系统状态改变

**动作**：系统状态改变，会触发额外动作

**迁移条件**：状态的迁移需要状态和事件匹配

**优点**：

**提高CPU 使用效率**  
话说我只要见到满篇都是delay_ms()的程序就会头疼，因为这些软件延时是对CPU资源的巨大浪费，宝贵的CPU机时都浪费在了NOP指令上。那种为了等待一个管脚电平跳变或者一个串口数据而岿然不动的程序也让我非常纠结，如果事件一直不发生，你要等到世界末日么？

把程序状态机化，这种情况就会明显改观，程序只需要用全局变量 记录下工作状态，就可以转头去干别的工作了，当然忙完那些活儿之后要再看看工作状态有没有变化。只要目标事件(定时未到、电平没跳变、串口数据没收完)还没发生，工作状态就不会改变，程序就一直重复着“查询—做任务—查询—做任务”这样的循环，这样CPU就闲不下来了。

这种处理方法的实质就是在程序等待事件的过程中间隔性地插入一些有意义的工作，好让CPU不是一直无谓地等待。  
**逻辑完备性**  
对于一个逻辑完备的反应式系统，不管什么样的事件组合，系统都能正确处理事件，而且系统自身的工作状态也一直处在可知可控的状态中。反过来，如果一个系统的逻辑功能不完备，在某些特定事件组合的驱动下，系统就会进入一个不可知不可控的状态，与设计者的意图相悖。使用状态机编程可以使程序的逻辑更加完善，从而减少BUG。  
  

**程序结构清晰**  
使用状态机编程可以使程序的结构更加清晰。

程序员最痛苦的事儿莫过于读别人写的代码。如果代码不是很规范，而且手里还没有流程图，读代码会让人晕了又晕，只有顺着程序一遍又一遍的看，很多遍之后才能隐约地明白程序大体的工作过程。有流程图会好一点，但是如果程序比较大，流程图也不会画得多详细，很多细节上的过程还是要从代码中理解。

相比之下，用状态机写的程序要好很多，拿一张标准的UML状态转换图，再配上一些简明的文字说明，程序中的各个要素一览无余。程序中有哪些状态，会发生哪些事件，状态机如何响应，响应之后跳转到哪个状态，这些都十分明朗，甚至许多动作细节都能从状态转换图中找到。可以毫不夸张的说，有了UML状态转换图，程序流程图写都不用写。

‍

#### vb2_plane

```c
struct vb2_plane {
    void            *mem_priv;          // 内存后端的私有数据（比如 vmalloc 的 page 指针）
    struct dma_buf  *dbuf;             // DMABUF 共享
    unsigned int    dbuf_mapped;
    unsigned int    bytesused;          // ★ 实际数据长度（驱动在完成时设置）
    unsigned int    length;             // 分配的 buffer 大小
    unsigned int    min_length;
    union {
        unsigned int    offset;        // MMAP 模式：mmap offset
        unsigned long   userptr;       // USERPTR 模式：用户空间地址
        int             fd;            // DMABUF 模式：文件描述符
    } m;
    unsigned int    data_offset;       // 数据在 plane 中的偏移（比如有头部时）
};
```

`struct vb2_plane`​ 是**描述一块内存的元信息**，它不包含数据本身

**真实的数据存在** **​`mem_priv`​**​ **所指的那块内存里**，而不在 `vb2_plane` 结构体本身。

![image](/images/v4l2/image-20260712172746-kiblg9e.png)

驱动程序初始化时，就构造了vb2_queue，这是"buffer的队列"，一开始里面没有"buffer"

APP调用ioctl VIDIOC_REQBUFS向驱动申请N个buffer

驱动程序分配n(n<=N)个vb2_buffer结构体，然后

	对于普通摄像头，还分配一个vb2_plane结构体、vb2_vmalloc_buf结构体，最后分配存数据的buffer

	对于多平面摄像头，给每个vb2_buffer分配多个"vb2_plane结构体、vb2_vmalloc_buf结构体、存数据的buffer"

‍

#### vb2\_v4l2\_buffer

```c
struct vb2_v4l2_buffer {
    struct vb2_buffer   vb2_buf;        // ★ 内嵌核心 buffer

    __u32               flags;          // V4L2_BUF_FLAG_*
    __u32               field;          // 逐行/隔行扫描
    struct v4l2_timecode timecode;      // 时间码
    __u32               sequence;       // ★ 帧序号（airspy 里 s->sequence++）
};
```

这是vb2层的顶层封装

```
用户空间 (userspace)
    struct v4l2_buffer           ← UAPI，ioctl 参数的定义
        ├── index
        ├── type
        ├── bytesused
        ├── flags
        ├── field
        ├── timestamp
        ├── memory
        ├── m.offset / m.userptr / m.fd
        └── length

          ▲ ioctl 进入内核时 copy_from_user
          │ ioctl 返回时   copy_to_user
          ▼

内核 vb2 层
    struct vb2_v4l2_buffer       ← V4L2 封装层，内核内部使用
        ├── vb2_buf              ← struct vb2_buffer （核心 buffer）
        ├── flags                ← 和 v4l2_buffer 的 flags 对应
        ├── field
        ├── timecode
        └── sequence

    struct vb2_buffer            ← 核心 buffer
        ├── index
        ├── planes[]
        └── state                ← 状态机（用户完全看不见）

    struct vb2_plane             ← 每个 plane 的描述
        ├── mem_priv             ← ★ 指向真正的数据内存
        ├── bytesused
        ├── length
        └── m.offset / ...
```

入队列流程：

	APP调用ioctl VIDIOC_QBUF

	驱动程序根据其index找到vb2_buffer

	把这个vb2_buffer放入链表vb2_queue.queued_list

硬件驱动接收到数据后，比如URB传输完成后：

	从链表vb2_queue.queued_list找到(但是不移除)vb2_buffer

	把硬件数据存入vb2_buffer

	把vb2_buffer放入链表vb2_queue.done_list

出队列流程：

	APP调用ioctl VIDIOC_DQBUF

	驱动程序从链表vb2_queue.done_list取出并移除第1个vb2_buffer

	驱动程序也把这个vb2_buffer从链表vb2_queue.queued_list移除

### videobuffer2的3个ops

![image](/images/v4l2/image-20260712200458-4m8mx9t.png)

在vb2管理队列中有三个ops

这三个结构体的作用为

`const struct vb2_buf_ops *buf_ops`：在用户空间、内核空间之间传递buffer信息，通过下面几个宏来调用

![image](/images/v4l2/image-20260712200649-xhmukte.png)

`const struct vb2_mem_ops *mem_ops`：分配内存用的回调函数，通过下面几个宏来调用

![image](/images/v4l2/image-20260712200718-glpdkzf.png)

`const struct vb2_ops *ops`：硬件相关的回调函数，通过下面几个宏来调用

![image](/images/v4l2/image-20260712200728-cap7kr2.png)

这3个ops的层次图如下：

![image](/images/v4l2/image-20260712201055-pkoxolj.png)

‍

‍

# 控制流media framework

‍

# 

‍

